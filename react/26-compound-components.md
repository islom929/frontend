# Bo'lim 26: Compound Components

> Compound Components — komponent guruhi bir **maxsus parent**ga bog'langan holda ishlaydigan pattern. Klassik HTML misol: `<select>` va `<option>` — `<option>` faqat `<select>` ichida ma'noga ega, parent state ni share qiladi. React'da bu pattern UI library'lari uchun (Tabs, Accordion, Select, Dialog) eng ko'p ishlatilgan dizayn pattern'i. Ikki implementation: **`React.Children` API + `cloneElement`** (legacy) va **Context-based** (modern). Bu fayl ikkalasini qamrab oladi va qachon qaysi tanlash kerakligini ko'rsatadi.

---

## Mundarija

- [Compound Components Nima va Muammo](#compound-components-nima-va-muammo)
- [Compound Components Pattern Fundamentals](#compound-components-pattern-fundamentals)
- [Implementation 1 — `React.Children` API + `cloneElement` (Legacy)](#implementation-1--reactchildren-api--cloneelement-legacy)
- [`React.Children.map` va `Children.toArray`](#reactchildrenmap-va-childrentoarray)
- [`cloneElement` — Props Injection](#cloneelement--props-injection)
- [`Children.only` va `Children.count`](#childrenonly-va-childrencount)
- [Implementation 2 — Context-Based (Modern)](#implementation-2--context-based-modern)
- [Real-World: Tabs Compound Component](#real-world-tabs-compound-component)
- [Real-World: Select / Dropdown Compound Component](#real-world-select--dropdown-compound-component)
- [Real-World: Accordion Compound Component](#real-world-accordion-compound-component)
- [Modern Compound vs Children API — Decision Guide](#modern-compound-vs-children-api--decision-guide)
- [TypeScript Patterns for Compound Components](#typescript-patterns-for-compound-components)
- [Accessibility — ARIA va Keyboard Navigation](#accessibility--aria-va-keyboard-navigation)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Compound Components Nima va Muammo

### Nazariya

**Compound Components** — bir nechta komponent **bitta logical unit** sifatida ishlaydigan pattern. Komponent'lar yakka holda ma'no bermaydi — ular **maxsus parent** bilan birga ishlatilishi shart. State va behavior parent tomonidan boshqariladi, children parent state'ni "consume" qiladi.

HTML'dan klassik misol:

```html
<select value="apple">
  <option value="apple">Apple</option>
  <option value="banana">Banana</option>
  <option value="cherry">Cherry</option>
</select>
```

`<option>` `<select>` tashqarisida ishlamaydi — `<select>` selected state'ni boshqaradi, `<option>` o'zining `value`'ni beradi va parent'ning selected state'ini ko'radi. Bu — fundamental Compound Components pattern.

React'da analog:

```tsx
<Select value="apple" onChange={setSelected}>
  <Select.Option value="apple">Apple</Select.Option>
  <Select.Option value="banana">Banana</Select.Option>
  <Select.Option value="cherry">Cherry</Select.Option>
</Select>
```

NIMA UCHUN bu pattern kerak: **flexible, declarative API**. Caller JSX strukturasini xohlagancha tuzadi (composition), parent state'ni boshqaradi:

```tsx
// Flexible — child tartibi va styling caller boshqaradi
<Tabs defaultValue="profile">
  <Tabs.List className="custom-tabs">
    <Tabs.Tab value="profile" icon={<UserIcon />}>Profile</Tabs.Tab>
    <Tabs.Tab value="settings" disabled>Settings</Tabs.Tab>
  </Tabs.List>
  
  <Divider />  {/* Custom element orasida */}
  
  <Tabs.Panel value="profile">Profile content</Tabs.Panel>
  <Tabs.Panel value="settings">Settings content</Tabs.Panel>
</Tabs>
```

**Muammo Compound Components hal qiladi:**

1. **Prop drilling** — parent state'ni har child'ga manual uzatish kerak emas.
2. **Implicit state sharing** — caller state mavjudligini ko'rmaydi (parent yashirin boshqaradi).
3. **Flexible composition** — caller har xil JSX struktura yozishi mumkin.
4. **Encapsulation** — parent o'z internal state'ini yashirin saqlaydi.

QANDAY ISHLAYDI: ikkita implementation strategiyasi:

**Strategy 1: `React.Children` API + `cloneElement`** (Legacy, Pre-Context era)

```tsx
function Tabs({ children, defaultValue }: { children: React.ReactNode; defaultValue: string }) {
  const [active, setActive] = useState(defaultValue);
  
  return React.Children.map(children, child => 
    React.cloneElement(child as React.ReactElement, { active, setActive })
  );
}
```

Parent har child'ga props inject qiladi. Lekin **faqat direct children** — nested struktura bilan ishlamaydi.

**Strategy 2: Context-Based** (Modern)

```tsx
const TabsContext = createContext<{ active: string; setActive: (v: string) => void } | null>(null);

function Tabs({ children, defaultValue }: { children: React.ReactNode; defaultValue: string }) {
  const [active, setActive] = useState(defaultValue);
  return (
    <TabsContext.Provider value={{ active, setActive }}>
      {children}
    </TabsContext.Provider>
  );
}

function Tab({ value, children }: { value: string; children: React.ReactNode }) {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error('Tab must be in <Tabs>');
  // ...
}
```

Parent Provider, child'lar `useContext` orqali state'ga kiradi. **Har chuqurlikdagi nested struktura ishlaydi**.

> **Versiya evolyutsiyasi (Compound Components):**
> - **Pre-R16.3 (2018):** `React.Children` API + `cloneElement` asosiy strategiya. Context API limited (`React.createContext` minimal, mostly legacy `contextTypes`).
> - **R16.3+:** Stable Context API (`createContext`/`Provider`/`Consumer`) — Compound Components Context-based variant'ga ko'cha boshladi.
> - **R16.8+ (Hooks):** `useContext` — Context consumption simpler. Modern Compound Components default Context-based.
> - **R19+:** `<Context value>` shorthand (Provider qisqartirildi), `use(context)` conditional reading. Compound Components yana ergonomic.

<details>
<summary><strong>Under the Hood</strong></summary>

JSX'da Compound Components — komponent **static method/property** sifatida nested:

```tsx
function Tabs(props: TabsProps) { /* ... */ }

function TabsList(props: TabsListProps) { /* ... */ }
function TabsTab(props: TabsTabProps) { /* ... */ }
function TabsPanel(props: TabsPanelProps) { /* ... */ }

// Static attachment
Tabs.List = TabsList;
Tabs.Tab = TabsTab;
Tabs.Panel = TabsPanel;

export default Tabs;
```

JSX `<Tabs.List>` — `Tabs.List` property'ni qidiradi va component sifatida render qiladi. JSX transform:

```tsx
<Tabs.List>...</Tabs.List>

// Transform:
React.createElement(Tabs.List, null, ...);
// Equivalent:
React.createElement(TabsList, null, ...);
```

Static property — `Tabs` namespace ostida child komponent'larni tashkil qiladi. Bu **API ergonomics**: `<Tabs.List>` (namespace clarity) vs alohida import (`<TabsList>`). Ikkalasi ham ishlaydi:

```tsx
// Namespace
import Tabs from './Tabs';
<Tabs.List>...</Tabs.List>

// Named imports
import { Tabs, TabsList, TabsTab, TabsPanel } from './Tabs';
<TabsList>...</TabsList>
```

Library tanlovi — Radix UI, Headless UI, shadcn/ui — **named imports** (tree-shaking yaxshi). Eski library'lar (Reach UI v1) namespace.

JSX type checking — `<Tabs.List>` bilan TypeScript tekshiradi `Tabs` da `List` property mavjudligini.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Sodda Compound Component (Card):

```tsx
import React from 'react';

function Card({ children }: { children: React.ReactNode }) {
  return <div className="card">{children}</div>;
}

function CardHeader({ children }: { children: React.ReactNode }) {
  return <div className="card-header">{children}</div>;
}

function CardBody({ children }: { children: React.ReactNode }) {
  return <div className="card-body">{children}</div>;
}

function CardFooter({ children }: { children: React.ReactNode }) {
  return <div className="card-footer">{children}</div>;
}

// Static attachment
Card.Header = CardHeader;
Card.Body = CardBody;
Card.Footer = CardFooter;

// Usage — flexible composition
<Card>
  <Card.Header>
    <h2>Profile</h2>
  </Card.Header>
  <Card.Body>
    <p>User profile content</p>
  </Card.Body>
  <Card.Footer>
    <button>Edit</button>
  </Card.Footer>
</Card>
```

Bu — **stateless** Compound Component. State sharing yo'q, faqat structural pattern. Asosiy benefit — namespace organization.

State-sharing Compound Component:

```tsx
// Tabs with state sharing — Modern (Context-based)
const TabsContext = React.createContext<{
  active: string;
  setActive: (value: string) => void;
} | null>(null);

function useTabs() {
  const ctx = React.useContext(TabsContext);
  if (!ctx) throw new Error('Tabs subcomponents must be used within <Tabs>');
  return ctx;
}

function Tabs({ 
  defaultValue, 
  children 
}: { 
  defaultValue: string; 
  children: React.ReactNode;
}) {
  const [active, setActive] = React.useState(defaultValue);
  return (
    <TabsContext.Provider value={{ active, setActive }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

function Tab({ value, children }: { value: string; children: React.ReactNode }) {
  const { active, setActive } = useTabs();
  return (
    <button
      className={active === value ? 'active' : ''}
      onClick={() => setActive(value)}
    >
      {children}
    </button>
  );
}

function TabPanel({ value, children }: { value: string; children: React.ReactNode }) {
  const { active } = useTabs();
  if (active !== value) return null;
  return <div className="tab-panel">{children}</div>;
}

Tabs.Tab = Tab;
Tabs.Panel = TabPanel;

// Usage
<Tabs defaultValue="overview">
  <div className="tabs-list">
    <Tabs.Tab value="overview">Overview</Tabs.Tab>
    <Tabs.Tab value="details">Details</Tabs.Tab>
  </div>
  <Tabs.Panel value="overview">Overview content</Tabs.Panel>
  <Tabs.Panel value="details">Details content</Tabs.Panel>
</Tabs>
```

Caller `<Tabs.Tab value="...">` chaqiradi va parent `Tabs` Context orqali state share qiladi. Implicit state sharing — caller `active`/`setActive` propslarini ko'rmaydi.

</details>

---

## Compound Components Pattern Fundamentals

### Nazariya

Compound Components pattern'ining asosiy elementlari:

1. **Parent Component** — state'ni boshqaradi, child'larga state'ni share qiladi (Context yoki cloneElement orqali).
2. **Child Components** — parent state'ni "consume" qiladi va o'z UI'ni render qiladi.
3. **Composition** — caller JSX strukturasini erkin tuzadi.
4. **Implicit state** — caller state'ni ko'rmaydi (encapsulation).

```
<Parent>                      ← state owner
  <Parent.SubComponent />     ← state consumer
  <Parent.SubComponent />     ← state consumer
</Parent>
```

**Ikki turdagi Compound Components:**

**Type 1: Static Compound** — state sharing yo'q, faqat structural organization.

```tsx
<Card>
  <Card.Header />
  <Card.Body />
  <Card.Footer />
</Card>
```

`Card`, `Card.Header`, `Card.Body`, `Card.Footer` — har biri mustaqil komponent. Hech qanday state share qilinmaydi. Faqat **namespace organization** maqsadida birlashtirilgan.

**Type 2: Stateful Compound** — state sharing bor, parent boshqaradi, child consume qiladi.

```tsx
<Tabs defaultValue="profile">
  <Tabs.List>
    <Tabs.Tab value="profile">Profile</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel value="profile">...</Tabs.Panel>
</Tabs>
```

`Tabs` `active` state'ni boshqaradi, `Tabs.Tab` clickda `setActive` chaqiradi, `Tabs.Panel` `active === value` tekshiradi.

NIMA UCHUN bu pattern foydali:

| Use case | Compound Components | Alternative |
|----------|---------------------|-------------|
| Tabs UI | `<Tabs>` + `<Tab>` + `<Panel>` | Single `<Tabs items={...}>` |
| Select dropdown | `<Select>` + `<Option>` | `<Select options={...}>` |
| Accordion | `<Accordion>` + `<Item>` + `<Content>` | `<Accordion items={...}>` |
| Modal | `<Modal>` + `<Header>` + `<Body>` + `<Footer>` | `<Modal title body footer>` |
| Form | `<Form>` + `<Field>` + `<Label>` | `<Form fields={...}>` |

Compound Components afzal:
- **Visual flexibility** — child tartibi, custom elementlar orasida
- **Slot pattern** — header, body, footer alohida customize qilinadi
- **Conditional rendering** — `{condition && <Tab>}` tabiiy

Single component (`items` prop) afzal:
- **Bulk data** — 100+ option'lar (Select)
- **Dynamic data** — server'dan kelgan list
- **Simple use case** — har item bir xil structure

QANDAY ISHLAYDI: parent React Element'larini render qiladi. Children'ga state'ni share qilish uchun:

- **Implicit (Context)** — Parent Provider, child useContext.
- **Explicit (cloneElement)** — Parent `React.Children.map` + `cloneElement` props inject.

<details>
<summary><strong>Under the Hood</strong></summary>

Compound Components vs Render Props vs Single Component memory comparison:

```
Static Compound (3 children):
  - Card Fiber
  - Card.Header Fiber
  - Card.Body Fiber
  - Card.Footer Fiber
  Total: 4 Fibers (children + parent)

Stateful Compound with Context (3 children):
  - Card Fiber (state)
  - Provider Fiber (Context)
  - Card.Header Fiber (Consumer)
  - Card.Body Fiber (Consumer)
  - Card.Footer Fiber (Consumer)
  Total: 5 Fibers

Render Props (1 provider, 1 consumer):
  - Provider Fiber (state)
  - Children function call result Fiber(s)
  Total: 2-3 Fibers

Single Component:
  - 1 Fiber
  - render renders all parts internally
  Total: 1 Fiber
```

Compound Components Fiber count yuqori, lekin **caller experience** yaxshi. Library design tanlovi.

`React.Children.count` — children Fiber count emas, balki **JSX children array length**:

```tsx
<Tabs>
  <Tab />
  <Tab />
  <Tab />
</Tabs>
// Children.count === 3

<Tabs>
  {true && <Tab />}
  {false && <Tab />}  // false — skip (boolean)
  null                // null — skip
  <Tab />
</Tabs>
// Children.count === 2 (true && <Tab /> = <Tab />, va oxirgi <Tab />)
// null va false hisoblanmaydi.
```

`Children.toArray` ham `null`/`undefined`/`boolean` ni olib tashlaydi (`Children.count` bilan bir xil iteration semantikasi):

```tsx
React.Children.toArray([
  <Tab />,
  null,
  false,
  <Tab />,
]);
// [<Tab />, <Tab />] — only valid elements
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Static Compound Component (Modal):

```tsx
function Modal({ 
  isOpen, 
  onClose, 
  children 
}: { 
  isOpen: boolean; 
  onClose: () => void; 
  children: React.ReactNode;
}) {
  if (!isOpen) return null;
  return (
    <div className="modal-backdrop" onClick={onClose}>
      <div className="modal" onClick={(e) => e.stopPropagation()}>
        {children}
      </div>
    </div>
  );
}

function ModalHeader({ children }: { children: React.ReactNode }) {
  return <header className="modal-header">{children}</header>;
}

function ModalBody({ children }: { children: React.ReactNode }) {
  return <div className="modal-body">{children}</div>;
}

function ModalFooter({ children }: { children: React.ReactNode }) {
  return <footer className="modal-footer">{children}</footer>;
}

Modal.Header = ModalHeader;
Modal.Body = ModalBody;
Modal.Footer = ModalFooter;

// Usage
<Modal isOpen={open} onClose={close}>
  <Modal.Header>
    <h2>Confirm Delete</h2>
  </Modal.Header>
  <Modal.Body>
    <p>Are you sure?</p>
  </Modal.Body>
  <Modal.Footer>
    <button onClick={close}>Cancel</button>
    <button onClick={confirm}>Delete</button>
  </Modal.Footer>
</Modal>
```

State'siz — har component independent. Caller xohlagan tartib va structure ishlatadi.

Stateful Compound (Toggle):

```tsx
const ToggleContext = React.createContext<{ on: boolean; toggle: () => void } | null>(null);

function useToggleContext() {
  const ctx = React.useContext(ToggleContext);
  if (!ctx) throw new Error('Toggle subcomponents must be inside <Toggle>');
  return ctx;
}

function Toggle({ 
  defaultOn = false, 
  children, 
  onChange 
}: { 
  defaultOn?: boolean; 
  children: React.ReactNode;
  onChange?: (on: boolean) => void;
}) {
  const [on, setOn] = React.useState(defaultOn);
  
  const toggle = React.useCallback(() => {
    setOn(prev => {
      const next = !prev;
      onChange?.(next);
      return next;
    });
  }, [onChange]);
  
  const value = React.useMemo(() => ({ on, toggle }), [on, toggle]);
  
  return (
    <ToggleContext.Provider value={value}>
      {children}
    </ToggleContext.Provider>
  );
}

function ToggleOn({ children }: { children: React.ReactNode }) {
  const { on } = useToggleContext();
  return on ? <>{children}</> : null;
}

function ToggleOff({ children }: { children: React.ReactNode }) {
  const { on } = useToggleContext();
  return !on ? <>{children}</> : null;
}

function ToggleButton(props: React.ButtonHTMLAttributes<HTMLButtonElement>) {
  const { toggle } = useToggleContext();
  return <button {...props} onClick={toggle} />;
}

Toggle.On = ToggleOn;
Toggle.Off = ToggleOff;
Toggle.Button = ToggleButton;

// Usage — flexible
<Toggle defaultOn={false} onChange={(on) => console.log('toggled:', on)}>
  <Toggle.On>
    <p>Light is ON</p>
  </Toggle.On>
  <Toggle.Off>
    <p>Light is OFF</p>
  </Toggle.Off>
  <Toggle.Button>Toggle Light</Toggle.Button>
</Toggle>
```

`Toggle.On` va `Toggle.Off` conditional render — parent `on` state'iga qarab. Caller faqat JSX struktura yozadi, state implicit boshqariladi.

</details>

---

## Implementation 1 — `React.Children` API + `cloneElement` (Legacy)

### Nazariya

Pre-Context era'da Compound Components **`React.Children` API** orqali implement qilinardi. Parent children'ni iterate qiladi va `cloneElement` orqali props inject qiladi.

```tsx
function Tabs({ children, defaultValue }: { children: React.ReactNode; defaultValue: string }) {
  const [active, setActive] = useState(defaultValue);
  
  return (
    <div className="tabs">
      {React.Children.map(children, child => {
        if (!React.isValidElement(child)) return child;
        return React.cloneElement(child, { active, setActive });
      })}
    </div>
  );
}

function Tab({ 
  value, 
  active, 
  setActive, 
  children 
}: { 
  value: string; 
  active?: string; 
  setActive?: (v: string) => void; 
  children: React.ReactNode;
}) {
  return (
    <button
      className={active === value ? 'active' : ''}
      onClick={() => setActive?.(value)}
    >
      {children}
    </button>
  );
}
```

QANDAY ISHLAYDI:

1. **Parent** state'ni boshqaradi.
2. **`React.Children.map`** har child uchun callback chaqiradi.
3. **`cloneElement`** child element'ni klonlaydi va qo'shimcha props beradi.
4. **Child** props (state) ni qabul qiladi va o'z UI'ni render qiladi.

NIMA UCHUN legacy: bu approach **direct children only** ishlaydi. Nested struktura — props inject qilinmaydi:

```tsx
<Tabs defaultValue="profile">
  <Tab value="profile">Profile</Tab>     {/* ✅ direct child — props inject */}
  <div>                                   {/* ⚠️ wrapper — break injection */}
    <Tab value="settings">Settings</Tab>  {/* ❌ nested — props inject yo'q */}
  </div>
</Tabs>
```

`Tabs` `<div>`'ga props inject qilmoqchi bo'ladi (lekin `<div>` bu prop'larni qabul qilmaydi). Ichidagi `<Tab>` props olmaydi.

**Workaround** — recursive `Children.map`:

```tsx
function Tabs({ children, defaultValue }: { children: React.ReactNode; defaultValue: string }) {
  const [active, setActive] = useState(defaultValue);
  
  function injectProps(elements: React.ReactNode): React.ReactNode {
    return React.Children.map(elements, child => {
      if (!React.isValidElement(child)) return child;
      
      // Tab element — inject props
      if (child.type === Tab) {
        return React.cloneElement(child, { active, setActive });
      }
      
      // Wrapper element — recurse into children
      if (child.props.children) {
        return React.cloneElement(child, {
          children: injectProps(child.props.children),
        });
      }
      
      return child;
    });
  }
  
  return <div className="tabs">{injectProps(children)}</div>;
}
```

Recursive — bir necha daraja chuqurlik. Lekin **performance overhead** (har render'da clone) va **complexity** (recursion logic). Modern Context-based alternativa har chuqurlikda ishlaydi va simple.

> **Versiya evolyutsiyasi (`cloneElement` Pattern):**
> - **Pre-R16.3 (2018):** `cloneElement` Compound Components uchun asosiy strategiya. Context API minimal.
> - **R16.3+:** Stable Context API, Compound Components Context-based variant'ga ko'chish boshlandi.
> - **R19+:** `cloneElement` deprecated emas, lekin community recommendation Context-based. `cloneElement` ko'pincha Library API'larda saqlanadi (backward compat).

<details>
<summary><strong>Under the Hood</strong></summary>

`React.cloneElement` source code (simplified):

```javascript
// react/src/ReactElement.js
function cloneElement(element, config, children) {
  // Original element copy
  const props = Object.assign({}, element.props);
  
  let key = element.key;
  let ref = element.ref;
  
  // Override props from config
  if (config != null) {
    if (hasValidRef(config)) {
      ref = config.ref;
    }
    if (hasValidKey(config)) {
      key = '' + config.key;
    }
    
    // Merge other props
    for (const propName in config) {
      if (
        config.hasOwnProperty(propName) &&
        propName !== 'key' && propName !== 'ref'
      ) {
        props[propName] = config[propName];
      }
    }
  }
  
  // Override children
  if (children) {
    props.children = children;
  }
  
  // Create new element with merged props
  return ReactElement(element.type, key, ref, ..., props);
}
```

`cloneElement` **shallow merge** — config'dagi prop'lar element prop'larini override qiladi. Children explicit beriladi.

`cloneElement` faqat **React Element** (JSX'dan natija) bilan ishlaydi. Boshqa qiymatlar (string, number, null) — `isValidElement(child) === false`.

```javascript
function isValidElement(object) {
  return (
    typeof object === 'object' &&
    object !== null &&
    object.$$typeof === REACT_ELEMENT_TYPE
  );
}
```

`$$typeof` symbol — React internal tag. JSX transform har element'ga qo'shadi.

Performance — `cloneElement` har render'da yangi Element obyekt yaratadi. Reconciler diff har render'da. Ko'p children bo'lsa — overhead.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Sodda RadioGroup `cloneElement` bilan:

```tsx
import React, { useState } from 'react';

interface RadioGroupProps {
  value: string;
  onChange: (value: string) => void;
  children: React.ReactNode;
}

function RadioGroup({ value, onChange, children }: RadioGroupProps) {
  return (
    <div className="radio-group" role="radiogroup">
      {React.Children.map(children, child => {
        if (!React.isValidElement(child)) return child;
        return React.cloneElement(child as React.ReactElement<RadioProps>, {
          checked: child.props.value === value,
          onChange,
        });
      })}
    </div>
  );
}

interface RadioProps {
  value: string;
  checked?: boolean;
  onChange?: (value: string) => void;
  children: React.ReactNode;
}

function Radio({ value, checked, onChange, children }: RadioProps) {
  return (
    <label className="radio">
      <input
        type="radio"
        value={value}
        checked={checked}
        onChange={() => onChange?.(value)}
      />
      <span>{children}</span>
    </label>
  );
}

RadioGroup.Item = Radio;

// Usage
function PaymentForm() {
  const [method, setMethod] = useState('card');
  
  return (
    <RadioGroup value={method} onChange={setMethod}>
      <RadioGroup.Item value="card">Credit Card</RadioGroup.Item>
      <RadioGroup.Item value="paypal">PayPal</RadioGroup.Item>
      <RadioGroup.Item value="bank">Bank Transfer</RadioGroup.Item>
    </RadioGroup>
  );
}
```

Bu yondashuv **direct children** uchun ishlaydi. Wrapper (`<div>`) qo'shilsa — sinmaydi.

`cloneElement` o'rniga `Children.map` + manual element yaratish:

```tsx
function RadioGroupAlternative({ value, onChange, children }: RadioGroupProps) {
  return (
    <div className="radio-group" role="radiogroup">
      {React.Children.map(children, child => {
        if (!React.isValidElement<RadioProps>(child)) return child;
        
        // Or: explicit re-create
        return (
          <Radio
            key={child.key ?? child.props.value}
            value={child.props.value}
            checked={child.props.value === value}
            onChange={onChange}
          >
            {child.props.children}
          </Radio>
        );
      })}
    </div>
  );
}
```

Manual re-create — type-safe (TypeScript inference yaxshi), lekin verbose. `cloneElement` — generic (har element type bilan ishlaydi).

</details>

---

## `React.Children.map` va `Children.toArray`

### Nazariya

`React.Children` API — children manipulation uchun helper'lar. JSX children — har xil turdagi qiymatlar bo'lishi mumkin (element, array, string, null, false). `React.Children` ularni xavfsiz iterate qilish va array'ga aylantirish uchun.

API'lar:

| Method | Maqsad | Return |
|--------|--------|--------|
| `Children.map(children, fn)` | Iterate va transform | Array of results |
| `Children.forEach(children, fn)` | Iterate (no return) | void |
| `Children.count(children)` | Count children | number |
| `Children.toArray(children)` | Filter va array'ga aylantirish | ReactElement[] |
| `Children.only(children)` | Validate single child | ReactElement |

**`Children.map` vs Native `Array.map`** — kritik farq:

```tsx
// Native — null/false bilan break
const children = [<Tab />, null, false, <Tab />];
children.map(c => /* iterate */);  // works, but null/false in result

// React.Children.map — auto-key, auto-skip
React.Children.map(children, c => /* iterate */);
// auto-skip null/undefined/boolean
// auto-generate key based on parent fragment depth
```

`Children.map` avtomatik:
- **Auto-key generation** — parent fragment depth + index
- **Skip falsy values** — `null`/`undefined`/`true`/`false` natijada result array'da bo'lmaydi (auto-filter)

**Aniq mexanizm (React source `mapIntoArray`):** `undefined`/`true`/`false` avval `null` ga aylantiriladi, keyin callback `null` qiymat bilan ham chaqiriladi (technically). Lekin callback qaytargan `null`/`undefined` natijada result array'ga qo'shilmaydi — shu sabab amaliy "skip" effekt'i. Callback `string`/`number`/`ReactElement` uchun mazmunli qiymat bilan chaqiriladi.

Foydalanuvchi nuqtai nazaridan: `Children.map(children, (c, i) => c)` filtersiz pass-through ham `null`/`false` qiymatlarini result array'dan olib tashlaydi.

**`Children.toArray`** — children'ni clean array'ga aylantiradi:

```tsx
const children = [<Tab key="1" />, null, false, [<Tab key="2" />, <Tab key="3" />]];
React.Children.toArray(children);
// [<Tab key="1" />, <Tab key="2" />, <Tab key="3" />]
// null va false olib tashlandi, nested array flatten qilindi
```

`toArray` `null`/`undefined`/`boolean` olib tashlaydi. Native `flatMap` o'rnida ishlatiladi.

NIMA UCHUN: native `arr.map` JSX children bilan ishlatib bo'lmasa: children `ReactNode` (string, number, ReactElement, fragment, array, null, boolean — har biri valid). Native `.map` `null`/`false`'da .map called on undefined error bermaydi (lekin children type narrow shart). `React.Children` API bu narrowing'ni ichida qiladi.

QANDAY ISHLAYDI:

```javascript
// React.Children.map (simplified)
function map(children, fn) {
  const result = [];
  forEachChildren(children, (child, idx) => {
    const transformed = fn(child, idx);
    if (transformed !== null && transformed !== undefined) {
      // Auto-key generation
      if (Array.isArray(transformed)) {
        transformed.forEach(item => result.push(applyAutoKey(item, idx)));
      } else {
        result.push(applyAutoKey(transformed, idx));
      }
    }
  });
  return result;
}
```

Auto-key — `$.0`, `$.1` prefix added to existing key (avoiding collision with user keys). Nested arrays flattened.

<details>
<summary><strong>Under the Hood</strong></summary>

`Children.map` source code structure (simplified):

```javascript
// react/src/ReactChildren.js
function mapChildren(children, fn, context) {
  if (children == null) return children;
  
  const result = [];
  let count = 0;
  
  mapIntoArray(children, result, '', '', fn);
  
  return result;
}

function mapIntoArray(children, array, escapedPrefix, nameSoFar, callback) {
  const type = typeof children;
  
  if (type === 'undefined' || type === 'boolean') {
    children = null;  // skip
  }
  
  let invokeCallback = false;
  
  if (children === null) {
    invokeCallback = true;
  } else {
    switch (type) {
      case 'string':
      case 'number':
        invokeCallback = true;
        break;
      case 'object':
        if (children.$$typeof === REACT_ELEMENT_TYPE) {
          invokeCallback = true;
        }
    }
  }
  
  if (invokeCallback) {
    const child = children;
    let mappedChild = callback(child, count);
    // Apply auto-key with escapedPrefix + index
    array.push(/* mapped with key */);
    return 1;
  }
  
  // Iterable children (array, iterator)
  let subtreeCount = 0;
  if (Array.isArray(children)) {
    children.forEach((child, i) => {
      subtreeCount += mapIntoArray(child, array, /* prefix */, /* name */, callback);
    });
  }
  
  return subtreeCount;
}
```

Algorithm — recursive walk through children, callback chaqirish, auto-key generation.

`Children.toArray` — `Children.map(children, c => c)` short-form. Lekin internal optimization — direct array creation:

```javascript
function toArray(children) {
  return mapChildren(children, child => child) || [];
}
```

Performance — `Children.toArray` siz native `[].concat(children)` ishlatish mumkin (children'ni flatten qilmaydi). Lekin iterator/Symbol.iterator children bilan crash. `Children.toArray` xavfsiz.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`Children.map` iteration:

```tsx
function NumberedList({ children }: { children: React.ReactNode }) {
  return (
    <ol>
      {React.Children.map(children, (child, index) => (
        <li>
          <span className="number">{index + 1}.</span>
          {child}
        </li>
      ))}
    </ol>
  );
}

<NumberedList>
  <p>First item</p>
  <p>Second item</p>
  <p>Third item</p>
</NumberedList>

// Output:
// <ol>
//   <li><span>1.</span><p>First item</p></li>
//   <li><span>2.</span><p>Second item</p></li>
//   <li><span>3.</span><p>Third item</p></li>
// </ol>
```

`Children.toArray` filtering:

```tsx
function FilteredList({ children }: { children: React.ReactNode }) {
  const childArray = React.Children.toArray(children);
  
  // Type narrow — only ReactElement (not strings, numbers)
  const elements = childArray.filter(React.isValidElement);
  
  return (
    <div>
      <p>Total: {elements.length}</p>
      {elements.map((el, i) => (
        <div key={i}>{el}</div>
      ))}
    </div>
  );
}

<FilteredList>
  <Item />
  {null}              {/* filtered out */}
  {false && <Item />} {/* filtered out (false) */}
  <Item />
  {"string"}          {/* string element — REMAINS in toArray result */}
</FilteredList>
// childArray.length === 3 (2 Item + 1 string)
// elements.length === 2 (after isValidElement filter)
```

Conditional injection based on position:

```tsx
function Stepper({ currentStep, children }: { currentStep: number; children: React.ReactNode }) {
  return (
    <div className="stepper">
      {React.Children.map(children, (child, index) => {
        if (!React.isValidElement(child)) return child;
        return React.cloneElement(child, {
          stepNumber: index + 1,
          isActive: index === currentStep,
          isCompleted: index < currentStep,
        });
      })}
    </div>
  );
}

interface StepProps {
  stepNumber?: number;
  isActive?: boolean;
  isCompleted?: boolean;
  children: React.ReactNode;
}

function Step({ stepNumber, isActive, isCompleted, children }: StepProps) {
  return (
    <div className={`step ${isActive ? 'active' : ''} ${isCompleted ? 'completed' : ''}`}>
      <span>{stepNumber}</span>
      {children}
    </div>
  );
}

// Usage
<Stepper currentStep={1}>
  <Step>Personal Info</Step>
  <Step>Address</Step>
  <Step>Confirmation</Step>
</Stepper>
```

</details>

---

## `cloneElement` — Props Injection

### Nazariya

`React.cloneElement` — mavjud React Element'ning kloni'ni yaratadi va qo'shimcha props bilan augment qiladi. Compound Components legacy implementation'ning asosiy primitive'i.

API:

```tsx
React.cloneElement(
  element: React.ReactElement,
  props?: object,
  ...children?: React.ReactNode[]
): React.ReactElement
```

Argumentlar:
- `element` — clone qilinadigan React Element
- `props` — yangi/override props
- `children` — children replace (optional)

Behavior:

1. **Shallow props merge** — original props + new props (new override).
2. **`key` va `ref` saqlanadi** (yoki override).
3. **Children replace** — agar berilsa, original children replaced.

```tsx
const original = <button className="primary" onClick={handleClick}>Click me</button>;

const cloned = React.cloneElement(original, {
  className: 'secondary',  // override
  disabled: true,          // new prop
});

// Result: <button className="secondary" onClick={handleClick} disabled>Click me</button>
```

`onClick` saqlandi (config'da yo'q), `className` override, `disabled` qo'shildi, children saqlandi.

NIMA UCHUN `cloneElement`:
- **Props injection** — Compound Components parent → child state share.
- **Behavior augmentation** — komponent atrofida wrapper logic (HOC alternative).
- **Library API** — flexible component wrapping.

QANDAY ISHLAYDI: `cloneElement` Element type'ni saqlaydi, faqat props ni `Object.assign`'lash. Original element o'zgarmaydi (immutable). Yangi element qaytariladi.

```tsx
function withTooltip(element: React.ReactElement, message: string) {
  return React.cloneElement(element, {
    title: message,  // HTML title attribute
    'data-tooltip': message,
  });
}

const button = <button>Hover me</button>;
const buttonWithTooltip = withTooltip(button, 'This is a tooltip');
```

**Cheklov'lar**:

1. **Faqat React Element** — `isValidElement` check shart.
2. **Direct children only** — nested struktura'ga inject qilmaydi.
3. **Props collision** — original props bilan collision (override behavior).
4. **`ref` callback re-attach** — har clone'da ref callback yangidan chaqirilishi mumkin.

> **Versiya evolyutsiyasi (`cloneElement`):**
> - **Pre-R16.8:** Compound Components va HOC alternative pattern uchun keng ishlatilgan.
> - **R16.8+:** Hooks va Context'larning paydo bo'lishi bilan kamroq ishlatiladi.
> - **R19+:** `cloneElement` deprecated emas, lekin React docs **modern alternative**'larni tavsiya qiladi: `Context` (state share), `as` prop (polymorphic component), render prop / children-as-function.

<details>
<summary><strong>Under the Hood</strong></summary>

`cloneElement` immutability:

```javascript
const elem1 = <button onClick={handler}>Click</button>;
const elem2 = React.cloneElement(elem1, { disabled: true });

console.log(elem1 === elem2); // false (different objects)
console.log(elem1.props === elem2.props); // false (different props)
console.log(elem1.props.onClick === elem2.props.onClick); // true (same handler reference)
```

Original element saqlanadi. Yangi element yangi `props` object — shallow.

`cloneElement` event handler merging anti-pattern:

```tsx
// ❌ Override silently — original onClick lost
React.cloneElement(child, { onClick: newHandler });

// ✅ Merge handlers
React.cloneElement(child, {
  onClick: (e) => {
    child.props.onClick?.(e);
    newHandler(e);
  },
});
```

`cloneElement` props merge — last-wins. Manual handler chaining shart.

`ref` cloneElement bilan:

```tsx
const original = <input ref={originalRef} />;
const cloned = React.cloneElement(original, { ref: newRef });

// Result — newRef ishlaydi, originalRef yo'qoladi
```

Multiple ref'lar uchun **ref forwarding** yoki **ref merge utility** kerak:

```tsx
function mergeRefs<T>(...refs: React.Ref<T>[]): React.RefCallback<T> {
  return (value) => {
    refs.forEach(ref => {
      if (typeof ref === 'function') {
        ref(value);
      } else if (ref) {
        (ref as React.RefObject<T | null>).current = value;
      }
    });
  };
}

const original = <input ref={refA} />;
const cloned = React.cloneElement(original, { ref: mergeRefs(refA, refB) });
// Both refs receive the value
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Tooltip wrapper:

```tsx
function Tooltip({ message, children }: { message: string; children: React.ReactElement }) {
  if (!React.isValidElement(children)) return children;
  
  return React.cloneElement(children, {
    title: message,
    'aria-label': message,
  });
}

// Usage
<Tooltip message="Submit the form">
  <button>Submit</button>
</Tooltip>
// Equivalent to: <button title="Submit the form" aria-label="Submit the form">Submit</button>
```

Slot filling:

```tsx
function Layout({ 
  header, 
  children, 
  footer 
}: { 
  header: React.ReactElement; 
  children: React.ReactNode; 
  footer: React.ReactElement;
}) {
  return (
    <div className="layout">
      {React.cloneElement(header, { className: 'app-header' })}
      <main>{children}</main>
      {React.cloneElement(footer, { className: 'app-footer' })}
    </div>
  );
}

<Layout 
  header={<header><h1>My App</h1></header>}
  footer={<footer><p>© 2026</p></footer>}
>
  <p>Main content</p>
</Layout>
```

Event handler merging:

```tsx
function withClickLogger(child: React.ReactElement, label: string) {
  return React.cloneElement(child, {
    onClick: (e: React.MouseEvent) => {
      console.log(`[${label}] clicked`, e);
      // Call original handler if exists
      child.props.onClick?.(e);
    },
  });
}

const myButton = <button onClick={() => alert('Original!')}>Click</button>;
const loggedButton = withClickLogger(myButton, 'submit-button');

// On click:
// 1. Logs "[submit-button] clicked"
// 2. Alerts "Original!"
```

Compound Component with cloneElement:

```tsx
function ButtonGroup({ 
  size = 'medium', 
  variant = 'primary',
  children 
}: { 
  size?: 'small' | 'medium' | 'large';
  variant?: 'primary' | 'secondary';
  children: React.ReactNode;
}) {
  return (
    <div className="button-group">
      {React.Children.map(children, child => {
        if (!React.isValidElement(child)) return child;
        return React.cloneElement(child as React.ReactElement<{ 
          size?: string; 
          variant?: string; 
        }>, {
          size: child.props.size ?? size,
          variant: child.props.variant ?? variant,
        });
      })}
    </div>
  );
}

interface ButtonProps {
  size?: 'small' | 'medium' | 'large';
  variant?: 'primary' | 'secondary';
  children: React.ReactNode;
  onClick?: () => void;
}

function Button({ size = 'medium', variant = 'primary', children, onClick }: ButtonProps) {
  return (
    <button className={`btn btn-${size} btn-${variant}`} onClick={onClick}>
      {children}
    </button>
  );
}

ButtonGroup.Item = Button;

// Usage
<ButtonGroup size="large" variant="secondary">
  <ButtonGroup.Item>Save</ButtonGroup.Item>
  <ButtonGroup.Item>Cancel</ButtonGroup.Item>
  <ButtonGroup.Item variant="primary">Delete</ButtonGroup.Item>
</ButtonGroup>
// Last button keeps its own variant="primary" (not overridden)
```

</details>

---

## `Children.only` va `Children.count`

### Nazariya

**`React.Children.only(children)`** — children faqat bitta ReactElement bo'lishi kerakligini validate qiladi. Aks holda runtime error.

```tsx
function Provider({ children }: { children: React.ReactElement }) {
  const onlyChild = React.Children.only(children);
  return React.cloneElement(onlyChild, { extraProp: 'value' });
}

<Provider>
  <Child />
</Provider>
// ✅ ishlaydi

<Provider>
  <Child1 />
  <Child2 />
</Provider>
// ❌ Error: React.Children.only expected to receive a single React element child
```

NIMA UCHUN: ba'zi pattern'lar bitta child shart — masalan `Tooltip` faqat bitta target element atrofida wrapper.

**`React.Children.count(children)`** — children sonini qaytaradi:

```tsx
React.Children.count(<><a /><b /><c /></>); // 3
React.Children.count(null); // 0
React.Children.count("hello"); // 1
```

`null`, `undefined`, `boolean` hisoblanmaydi. String/number 1 element sifatida.

NIMA UCHUN: validation, conditional rendering, layout decisions:

```tsx
function Modal({ children }: { children: React.ReactNode }) {
  const childCount = React.Children.count(children);
  
  if (childCount === 0) return null;
  if (childCount === 1) return <SimpleModal>{children}</SimpleModal>;
  return <ComplexModal>{children}</ComplexModal>;
}
```

QANDAY ISHLAYDI:

```javascript
// React.Children.only (simplified)
function only(children) {
  if (!isValidElement(children)) {
    throw new Error('React.Children.only expected to receive a single React element child');
  }
  return children;
}

// React.Children.count (simplified)
function count(children) {
  let n = 0;
  mapChildren(children, () => { n++; });
  return n;
}
```

`only` array'ni accept qilmaydi (faqat single element). `Children.count` `null`/`undefined`/`boolean`'ni skip qiladi (mapChildren ichida).

<details>
<summary><strong>Under the Hood</strong></summary>

`Children.only` xato message:

```javascript
function only(children) {
  if (!isValidElement(children)) {
    throw new Error(
      'React.Children.only expected to receive a single React element child.'
    );
  }
  return children;
}
```

`isValidElement` — `children.$$typeof === REACT_ELEMENT_TYPE`. Array — invalid (array `$$typeof` yo'q).

`Children.count` — `mapChildren` orqali iterate qiladi va counter increment qiladi. `null`/`boolean`/`undefined` skip (mapChildren return false on those). Strings/numbers — count'da hisoblanadi:

```javascript
React.Children.count("hello"); // 1
React.Children.count(123); // 1
React.Children.count(true); // 0
React.Children.count(false); // 0
React.Children.count(null); // 0
```

R19+ nominal `null` count'da:

```javascript
React.Children.count([null, <a />, null]); // 1 (only <a />)
```

Test for "have any children":

```tsx
const hasChildren = React.Children.count(children) > 0;
// vs
const hasChildren = !!React.Children.toArray(children).length;
```

Ikkalasi ham ishlaydi. `count` simpler.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`Children.only` Tooltip:

```tsx
interface TooltipProps {
  message: string;
  children: React.ReactElement;
}

function Tooltip({ message, children }: TooltipProps) {
  // Only single child allowed
  React.Children.only(children);
  
  return React.cloneElement(children, {
    title: message,
    'aria-label': message,
  });
}

// ✅ OK
<Tooltip message="Submit"><button>Submit</button></Tooltip>

// ❌ Throws: React.Children.only expected to receive a single React element child
<Tooltip message="Submit">
  <button>Submit</button>
  <button>Cancel</button>
</Tooltip>
```

`Children.count` conditional rendering:

```tsx
function Toolbar({ children }: { children: React.ReactNode }) {
  const count = React.Children.count(children);
  
  if (count === 0) return null;
  
  return (
    <div className="toolbar">
      <span className="toolbar-count">{count} items</span>
      <div className="toolbar-items">{children}</div>
    </div>
  );
}

<Toolbar>
  <Button>Save</Button>
  <Button>Cancel</Button>
  <Button>Delete</Button>
</Toolbar>
// "3 items"
```

Validation pattern:

```tsx
interface CarouselProps {
  children: React.ReactNode;
  minItems?: number;
}

function Carousel({ children, minItems = 2 }: CarouselProps) {
  const count = React.Children.count(children);
  
  if (count < minItems) {
    console.warn(`Carousel requires at least ${minItems} items, got ${count}`);
    return null;
  }
  
  return (
    <div className="carousel">
      {React.Children.map(children, (child, index) => (
        <div className="carousel-slide" key={index}>
          {child}
        </div>
      ))}
    </div>
  );
}
```

</details>

---

## Implementation 2 — Context-Based (Modern)

### Nazariya

Modern Compound Components Context API orqali implement qilinadi. Parent state'ni Context Provider qiladi, child'lar `useContext` orqali consume qiladi.

```tsx
const TabsContext = React.createContext<{
  active: string;
  setActive: (value: string) => void;
} | null>(null);

function useTabsContext() {
  const ctx = React.useContext(TabsContext);
  if (!ctx) {
    throw new Error('Tabs subcomponents must be rendered inside <Tabs>');
  }
  return ctx;
}

function Tabs({ defaultValue, children }: { defaultValue: string; children: React.ReactNode }) {
  const [active, setActive] = useState(defaultValue);
  const value = useMemo(() => ({ active, setActive }), [active]);
  
  return (
    <TabsContext.Provider value={value}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

function Tab({ value, children }: { value: string; children: React.ReactNode }) {
  const { active, setActive } = useTabsContext();
  return (
    <button
      className={active === value ? 'active' : ''}
      onClick={() => setActive(value)}
    >
      {children}
    </button>
  );
}

function TabPanel({ value, children }: { value: string; children: React.ReactNode }) {
  const { active } = useTabsContext();
  if (active !== value) return null;
  return <div className="tab-panel">{children}</div>;
}

Tabs.Tab = Tab;
Tabs.Panel = TabPanel;
```

QANDAY ISHLAYDI:

1. `Tabs` (parent) Context Provider — state ekspozs.
2. `Tab` va `TabPanel` (children) — `useContext(TabsContext)` orqali state consume.
3. **Har chuqurlikda nested** — Context React tree bo'ylab uzatadi.

NIMA UCHUN modern variant afzal:

| Aspect | `cloneElement` | Context |
|--------|----------------|---------|
| Direct children only | ✅ | ❌ |
| Nested struktura | ❌ | ✅ |
| TypeScript inference | Manual | Auto |
| Performance (re-render) | All children clone | Selected consumers |
| Custom hook integration | Murakkab | `useContext` natural |
| Debugging | DevTools cluttered | Clean tree |

**Custom hook pattern** — strict consumer hook:

```tsx
function useTabsContext() {
  const ctx = React.useContext(TabsContext);
  if (!ctx) {
    throw new Error(
      'Tabs subcomponent (Tabs.Tab, Tabs.Panel) must be used inside <Tabs>'
    );
  }
  return ctx;
}
```

Strict pattern — Provider tashqarida ishlatilsa runtime error. Default value `null` o'rniga `undefined` ham OK (TypeScript narrow `if (!ctx)`).

> **Versiya evolyutsiyasi (Modern Compound Components):**
> - **R16.3 (2018):** Stable `createContext` API.
> - **R16.8 (2019):** `useContext` hook — Context-based pattern dominant.
> - **R19 (2024):** `<TabsContext value={value}>` shorthand (`.Provider` qisqartirildi). `use(TabsContext)` conditional reading (cross-ref [`19-usecontext.md`](19-usecontext.md), [`23-r19-hooks.md`](23-r19-hooks.md)).

<details>
<summary><strong>Under the Hood</strong></summary>

Context-based Compound Components Fiber tree:

```
<Tabs defaultValue="profile">
  <TabList>
    <Tab value="profile" />
    <Tab value="settings" />
  </TabList>
  <TabPanel value="profile">...</TabPanel>
</Tabs>

Fiber tree:
- Tabs (state owner)
  ├── TabsContext.Provider
  │   ├── TabList
  │   │   ├── Tab (consumer 1)
  │   │   └── Tab (consumer 2)
  │   └── TabPanel (consumer 3)
```

Provider value o'zgarsa — barcha consumer'lar re-render. Lekin React optimization:

- `value` reference o'zgarmasa — bailout (Object.is comparison).
- `useMemo` Provider'da value memoize qilish shart.

```tsx
// ❌ Anti-pattern
function Tabs() {
  const [active, setActive] = useState('a');
  return (
    <TabsContext.Provider value={{ active, setActive }}>  {/* har render new object */}
      ...
    </TabsContext.Provider>
  );
}

// ✅ Memoized
function Tabs() {
  const [active, setActive] = useState('a');
  const value = useMemo(() => ({ active, setActive }), [active]);
  return (
    <TabsContext.Provider value={value}>
      ...
    </TabsContext.Provider>
  );
}
```

`setActive` `useState` setter — har render bir xil reference (cross-ref [`12-state-and-usestate.md`](12-state-and-usestate.md)). `value` faqat `active` o'zgarganda yangilanadi.

React Compiler (1.0 stable, R19.1+ bilan 2025-aprel; R17/18/19 mos) — auto-memoization (cross-ref [`31-react-compiler.md`](31-react-compiler.md)). Manual `useMemo` kerak emas (Compiler infer qiladi).

**Splitting Context** — performance optimization:

```tsx
// Single Context — har consumer state value o'zgarganda re-render
const TabsContext = createContext({ active, setActive });

// Splitted — state va dispatch alohida
const TabsStateContext = createContext({ active });
const TabsDispatchContext = createContext({ setActive });

function Tab() {
  // Faqat state'ga subscribe (active o'zgarsa re-render)
  const { active } = useContext(TabsStateContext);
}

function TabButton() {
  // Faqat dispatch'ga subscribe (state o'zgarsa re-render emas)
  const { setActive } = useContext(TabsDispatchContext);
}
```

(Cross-ref [`19-usecontext.md`](19-usecontext.md) — Splitting Contexts).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Counter Compound Component:

```tsx
import React, { createContext, useContext, useState, useMemo } from 'react';

interface CounterContextValue {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
}

const CounterContext = createContext<CounterContextValue | null>(null);

function useCounterContext(componentName: string): CounterContextValue {
  const ctx = useContext(CounterContext);
  if (!ctx) {
    throw new Error(`<${componentName}> must be used within <Counter>`);
  }
  return ctx;
}

function Counter({ 
  initialValue = 0, 
  children 
}: { 
  initialValue?: number; 
  children: React.ReactNode;
}) {
  const [count, setCount] = useState(initialValue);
  
  const value = useMemo<CounterContextValue>(() => ({
    count,
    increment: () => setCount(c => c + 1),
    decrement: () => setCount(c => c - 1),
    reset: () => setCount(initialValue),
  }), [count, initialValue]);
  
  return (
    <CounterContext.Provider value={value}>
      <div className="counter">{children}</div>
    </CounterContext.Provider>
  );
}

function CounterValue() {
  const { count } = useCounterContext('Counter.Value');
  return <span className="counter-value">{count}</span>;
}

function CounterIncrement(props: React.ButtonHTMLAttributes<HTMLButtonElement>) {
  const { increment } = useCounterContext('Counter.Increment');
  return <button {...props} onClick={increment} />;
}

function CounterDecrement(props: React.ButtonHTMLAttributes<HTMLButtonElement>) {
  const { decrement } = useCounterContext('Counter.Decrement');
  return <button {...props} onClick={decrement} />;
}

function CounterReset(props: React.ButtonHTMLAttributes<HTMLButtonElement>) {
  const { reset } = useCounterContext('Counter.Reset');
  return <button {...props} onClick={reset} />;
}

Counter.Value = CounterValue;
Counter.Increment = CounterIncrement;
Counter.Decrement = CounterDecrement;
Counter.Reset = CounterReset;

// Usage — flexible composition
<Counter initialValue={10}>
  <div className="counter-controls">
    <Counter.Decrement>-</Counter.Decrement>
    <Counter.Value />
    <Counter.Increment>+</Counter.Increment>
  </div>
  <Counter.Reset>Reset</Counter.Reset>
</Counter>

// Or different layout
<Counter>
  <Counter.Increment>Add 1</Counter.Increment>
  <p>
    Current count: <Counter.Value />
  </p>
</Counter>
```

Children'lar har xil chuqurlikda joylashishi mumkin — Context tree bo'ylab uzatadi.

</details>

---

## Real-World: Tabs Compound Component

### Nazariya

Tabs — eng klassik Compound Components misol. UI library'larda (Radix UI, Headless UI, shadcn/ui, Material UI) hammasida bor.

API design:

```tsx
<Tabs defaultValue="profile" onChange={handleChange}>
  <Tabs.List aria-label="Account settings">
    <Tabs.Tab value="profile">Profile</Tabs.Tab>
    <Tabs.Tab value="security">Security</Tabs.Tab>
    <Tabs.Tab value="billing" disabled>Billing</Tabs.Tab>
  </Tabs.List>
  
  <Tabs.Panel value="profile">Profile content</Tabs.Panel>
  <Tabs.Panel value="security">Security content</Tabs.Panel>
  <Tabs.Panel value="billing">Billing content</Tabs.Panel>
</Tabs>
```

Asosiy elementlar:

1. **`Tabs`** — root, state owner.
2. **`Tabs.List`** — Tab buttons container (semantic `<div role="tablist">`).
3. **`Tabs.Tab`** — Individual tab button.
4. **`Tabs.Panel`** — Tab content (faqat active panel render qilinadi).

State:
- `active: string` — joriy tab value.
- `setActive: (value: string) => void` — change handler.

NIMA UCHUN bu pattern: caller layout va styling'ni o'zi boshqaradi:

```tsx
// Vertical tabs
<Tabs defaultValue="profile">
  <div className="vertical-layout">
    <Tabs.List orientation="vertical">
      <Tabs.Tab value="profile">Profile</Tabs.Tab>
    </Tabs.List>
    <Tabs.Panel value="profile">...</Tabs.Panel>
  </div>
</Tabs>

// Horizontal with separator
<Tabs defaultValue="profile">
  <Tabs.List>
    <Tabs.Tab value="profile">Profile</Tabs.Tab>
  </Tabs.List>
  <hr />
  <Tabs.Panel value="profile">...</Tabs.Panel>
</Tabs>
```

Single component variant (flexibility yo'q):

```tsx
// Less flexible — predefined structure
<Tabs
  items={[
    { value: 'profile', label: 'Profile', content: <ProfileContent /> },
    { value: 'security', label: 'Security', content: <SecurityContent /> },
  ]}
  defaultValue="profile"
/>
```

QANDAY ISHLAYDI:

1. `Tabs` Provider Context (`active`, `setActive`).
2. `Tabs.Tab value="profile"` click → `setActive('profile')`.
3. `Tabs.Panel value="profile"` — `active === 'profile'` bo'lsa render.

Controlled vs Uncontrolled:

```tsx
// Uncontrolled — internal state
<Tabs defaultValue="profile">...</Tabs>

// Controlled — external state
<Tabs value={activeTab} onChange={setActiveTab}>...</Tabs>
```

Library'larda ikkalasi qo'llab-quvvatlanadi (cross-ref [`14-lifting-and-controlled.md`](14-lifting-and-controlled.md)).

<details>
<summary><strong>Under the Hood</strong></summary>

Tabs accessibility — ARIA roles:

```html
<!-- Required ARIA structure -->
<div role="tablist">
  <button role="tab" aria-selected="true" aria-controls="panel-1" id="tab-1">Tab 1</button>
  <button role="tab" aria-selected="false" aria-controls="panel-2" id="tab-2">Tab 2</button>
</div>

<div role="tabpanel" id="panel-1" aria-labelledby="tab-1">Content 1</div>
<div role="tabpanel" id="panel-2" aria-labelledby="tab-2" hidden>Content 2</div>
```

Keyboard navigation:

- **Arrow Left/Right** — focus next/prev tab
- **Home/End** — focus first/last tab
- **Enter/Space** — activate tab (if not auto-activate)
- **Tab** — leave tablist (next focusable)

`useId` (R18+, cross-ref [`22-concurrent-hooks.md`](22-concurrent-hooks.md)) — unique IDs SSR-safe.

Internal Context value structure:

```tsx
interface TabsContextValue {
  active: string;
  setActive: (value: string) => void;
  baseId: string;  // for ARIA IDs
  registerTab: (value: string, ref: HTMLButtonElement) => void;
  tabRefs: React.RefObject<Map<string, HTMLButtonElement>>;
}
```

Tab registry — keyboard navigation uchun (focus next/prev tab).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

To'liq Tabs implementation:

```tsx
import React, { createContext, useContext, useState, useId, useMemo, useRef, useCallback } from 'react';

interface TabsContextValue {
  active: string;
  setActive: (value: string) => void;
  baseId: string;
  registerTab: (value: string, ref: HTMLButtonElement | null) => void;
  focusTab: (value: string) => void;
  getTabValues: () => string[];
}

const TabsContext = createContext<TabsContextValue | null>(null);

function useTabsContext(componentName: string): TabsContextValue {
  const ctx = useContext(TabsContext);
  if (!ctx) {
    throw new Error(`<${componentName}> must be used inside <Tabs>`);
  }
  return ctx;
}

interface TabsProps {
  defaultValue: string;
  value?: string;
  onChange?: (value: string) => void;
  children: React.ReactNode;
}

export function Tabs({ defaultValue, value: controlledValue, onChange, children }: TabsProps) {
  const [internalValue, setInternalValue] = useState(defaultValue);
  const isControlled = controlledValue !== undefined;
  const active = isControlled ? controlledValue : internalValue;
  
  const baseId = useId();
  const tabRefs = useRef<Map<string, HTMLButtonElement>>(new Map());
  
  const setActive = useCallback((newValue: string) => {
    if (!isControlled) setInternalValue(newValue);
    onChange?.(newValue);
  }, [isControlled, onChange]);
  
  const registerTab = useCallback((tabValue: string, ref: HTMLButtonElement | null) => {
    if (ref) {
      tabRefs.current.set(tabValue, ref);
    } else {
      tabRefs.current.delete(tabValue);
    }
  }, []);
  
  const focusTab = useCallback((tabValue: string) => {
    tabRefs.current.get(tabValue)?.focus();
  }, []);
  
  const getTabValues = useCallback(() => Array.from(tabRefs.current.keys()), []);
  
  const contextValue = useMemo<TabsContextValue>(() => ({
    active,
    setActive,
    baseId,
    registerTab,
    focusTab,
    getTabValues,
  }), [active, setActive, baseId, registerTab, focusTab, getTabValues]);
  
  return (
    <TabsContext.Provider value={contextValue}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

interface TabListProps extends React.HTMLAttributes<HTMLDivElement> {
  children: React.ReactNode;
  orientation?: 'horizontal' | 'vertical';
}

export function TabList({ children, orientation = 'horizontal', ...rest }: TabListProps) {
  return (
    <div
      role="tablist"
      aria-orientation={orientation}
      className={`tab-list tab-list-${orientation}`}
      {...rest}
    >
      {children}
    </div>
  );
}

interface TabProps {
  value: string;
  children: React.ReactNode;
  disabled?: boolean;
}

export function Tab({ value, children, disabled }: TabProps) {
  const { active, setActive, baseId, registerTab, focusTab, getTabValues } = useTabsContext('Tabs.Tab');
  const buttonRef = useRef<HTMLButtonElement>(null);
  const isActive = active === value;
  
  React.useEffect(() => {
    registerTab(value, buttonRef.current);
    return () => registerTab(value, null);
  }, [value, registerTab]);
  
  const handleKeyDown = (e: React.KeyboardEvent<HTMLButtonElement>) => {
    const values = getTabValues();
    const currentIndex = values.indexOf(value);
    
    let nextIndex = currentIndex;
    if (e.key === 'ArrowRight' || e.key === 'ArrowDown') {
      nextIndex = (currentIndex + 1) % values.length;
    } else if (e.key === 'ArrowLeft' || e.key === 'ArrowUp') {
      nextIndex = (currentIndex - 1 + values.length) % values.length;
    } else if (e.key === 'Home') {
      nextIndex = 0;
    } else if (e.key === 'End') {
      nextIndex = values.length - 1;
    } else {
      return;
    }
    
    e.preventDefault();
    const nextValue = values[nextIndex];
    setActive(nextValue);
    focusTab(nextValue);
  };
  
  return (
    <button
      ref={buttonRef}
      role="tab"
      id={`${baseId}-tab-${value}`}
      aria-selected={isActive}
      aria-controls={`${baseId}-panel-${value}`}
      tabIndex={isActive ? 0 : -1}
      disabled={disabled}
      className={`tab ${isActive ? 'active' : ''}`}
      onClick={() => setActive(value)}
      onKeyDown={handleKeyDown}
    >
      {children}
    </button>
  );
}

interface TabPanelProps {
  value: string;
  children: React.ReactNode;
}

export function TabPanel({ value, children }: TabPanelProps) {
  const { active, baseId } = useTabsContext('Tabs.Panel');
  const isActive = active === value;
  
  return (
    <div
      role="tabpanel"
      id={`${baseId}-panel-${value}`}
      aria-labelledby={`${baseId}-tab-${value}`}
      hidden={!isActive}
      className="tab-panel"
    >
      {isActive && children}
    </div>
  );
}

Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;

// Usage
function AccountSettings() {
  return (
    <Tabs defaultValue="profile">
      <Tabs.List aria-label="Account settings">
        <Tabs.Tab value="profile">Profile</Tabs.Tab>
        <Tabs.Tab value="security">Security</Tabs.Tab>
        <Tabs.Tab value="billing" disabled>Billing</Tabs.Tab>
      </Tabs.List>
      
      <Tabs.Panel value="profile">
        <h3>Profile Settings</h3>
        <p>Edit your profile here.</p>
      </Tabs.Panel>
      <Tabs.Panel value="security">
        <h3>Security Settings</h3>
        <p>Change your password.</p>
      </Tabs.Panel>
      <Tabs.Panel value="billing">
        <h3>Billing</h3>
      </Tabs.Panel>
    </Tabs>
  );
}
```

Bu **production-grade** Tabs:
- Controlled/Uncontrolled
- Keyboard navigation (Arrow keys, Home, End)
- ARIA roles + IDs (useId R18+)
- Tab registry (focus next/prev)
- Disabled state
- TypeScript safe

</details>

---

## Real-World: Select / Dropdown Compound Component

### Nazariya

Select (custom dropdown) — Compound Components classic use case. Native `<select>` cheklov'lari (custom styling, search, multi-select) tufayli custom implementation.

API design:

```tsx
<Select value={selected} onChange={setSelected}>
  <Select.Trigger>
    <Select.Value placeholder="Select option..." />
    <Select.Icon />
  </Select.Trigger>
  
  <Select.Content>
    <Select.Item value="apple">Apple</Select.Item>
    <Select.Item value="banana">Banana</Select.Item>
    <Select.Group label="Citrus">
      <Select.Item value="orange">Orange</Select.Item>
      <Select.Item value="lemon">Lemon</Select.Item>
    </Select.Group>
  </Select.Content>
</Select>
```

Asosiy elementlar:

1. **`Select`** — root, state owner.
2. **`Select.Trigger`** — Button, dropdown'ni ochadi.
3. **`Select.Value`** — Selected value display.
4. **`Select.Content`** — Dropdown content (open bo'lganda).
5. **`Select.Item`** — Individual option.
6. **`Select.Group`** — Optional grouping (`<optgroup>` analog).

State:
- `value: string` — selected value
- `setValue: (value: string) => void`
- `isOpen: boolean` — dropdown open
- `setIsOpen: (open: boolean) => void`

Behavior:

- Trigger click — toggle open
- Item click — select + close
- Outside click — close
- Escape key — close
- Arrow keys — focus item
- Enter — select focused item

NIMA UCHUN custom Select: native `<select>`:
- Custom styling cheklangan (browser default options dropdown)
- Custom item content yo'q (faqat string)
- Search/filter yo'q
- Multi-level grouping cheklangan

Custom dropdown — full control. Lekin **accessibility ko'p ishlash kerak** (ARIA, keyboard, focus management).

Production library'lar (Radix UI, Headless UI, shadcn/ui) — accessibility-first Select.

QANDAY ISHLAYDI:

1. `Select.Trigger` click — `setIsOpen(true)`.
2. `Select.Content` — `isOpen` bo'lsa render (Portal — z-index, overflow).
3. `Select.Item` click — `setValue(value)` + `setIsOpen(false)`.
4. Document click outside — `setIsOpen(false)` (cross-ref [`24-custom-hooks.md`](24-custom-hooks.md) `useOnClickOutside`).
5. Escape key — `setIsOpen(false)`.

<details>
<summary><strong>Under the Hood</strong></summary>

Select ARIA structure:

```html
<button 
  role="combobox" 
  aria-expanded="true" 
  aria-haspopup="listbox" 
  aria-controls="select-content"
>
  Apple
</button>

<div role="listbox" id="select-content">
  <div role="option" aria-selected="true" tabindex="-1">Apple</div>
  <div role="option" aria-selected="false" tabindex="-1">Banana</div>
</div>
```

Keyboard:
- **Tab** — focus trigger
- **Enter/Space** — open listbox
- **Arrow Up/Down** — navigate items
- **Enter** — select focused item
- **Escape** — close

Portal usage — Select.Content body'ga render (z-index, overflow:hidden parent qutulish):

```tsx
import { createPortal } from 'react-dom';

function SelectContent({ children }: { children: React.ReactNode }) {
  const { isOpen } = useSelectContext();
  if (!isOpen) return null;
  
  return createPortal(
    <div role="listbox" className="select-content">
      {children}
    </div>,
    document.body
  );
}
```

Cross-ref [`28-portals.md`](28-portals.md) — Portal pattern.

Position calculation — trigger button bo'yicha (`useLayoutEffect` — paint'dan oldin, no flicker; cross-ref [`17-uselayouteffect.md`](17-uselayouteffect.md)):

```tsx
useLayoutEffect(() => {
  if (isOpen && triggerRef.current && contentRef.current) {
    const rect = triggerRef.current.getBoundingClientRect();
    contentRef.current.style.top = `${rect.bottom}px`;
    contentRef.current.style.left = `${rect.left}px`;
    contentRef.current.style.width = `${rect.width}px`;
  }
}, [isOpen]);
```

`useEffect` ishlatilsa, content avval default position'da paint qilinadi (`top: 0, left: 0`), keyin to'g'ri position'ga "sakraydi" (flicker). `useLayoutEffect` paint'dan oldin position'ni o'rnatadi.

Modern alternative — `@floating-ui/react` (positioning library, viewport-aware, auto-flip, collision detection).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

To'liq Select implementation (simplified, no Portal):

```tsx
import React, { createContext, useContext, useState, useId, useMemo, useRef, useEffect, useCallback } from 'react';

interface SelectContextValue {
  value: string;
  setValue: (value: string) => void;
  isOpen: boolean;
  setIsOpen: (open: boolean) => void;
  baseId: string;
  triggerRef: React.RefObject<HTMLButtonElement>;
  registerItem: (value: string, label: string) => void;
  selectedLabel: string;
}

const SelectContext = createContext<SelectContextValue | null>(null);

function useSelectContext(componentName: string) {
  const ctx = useContext(SelectContext);
  if (!ctx) throw new Error(`<${componentName}> must be inside <Select>`);
  return ctx;
}

export function Select({ 
  value: controlledValue,
  defaultValue,
  onChange,
  children 
}: { 
  value?: string;
  defaultValue?: string;
  onChange?: (value: string) => void;
  children: React.ReactNode;
}) {
  const [internalValue, setInternalValue] = useState(defaultValue ?? '');
  const isControlled = controlledValue !== undefined;
  const value = isControlled ? controlledValue : internalValue;
  
  const [isOpen, setIsOpen] = useState(false);
  const baseId = useId();
  const triggerRef = useRef<HTMLButtonElement>(null);
  const itemsMap = useRef<Map<string, string>>(new Map());
  
  const setValue = useCallback((newValue: string) => {
    if (!isControlled) setInternalValue(newValue);
    onChange?.(newValue);
    setIsOpen(false);
  }, [isControlled, onChange]);
  
  const registerItem = useCallback((itemValue: string, label: string) => {
    itemsMap.current.set(itemValue, label);
  }, []);
  
  const selectedLabel = itemsMap.current.get(value) ?? '';
  
  // Click outside
  useEffect(() => {
    if (!isOpen) return;
    
    const handler = (e: MouseEvent) => {
      const target = e.target as Node;
      if (triggerRef.current && !triggerRef.current.contains(target)) {
        const content = document.getElementById(`${baseId}-content`);
        if (!content?.contains(target)) {
          setIsOpen(false);
        }
      }
    };
    
    document.addEventListener('mousedown', handler);
    return () => document.removeEventListener('mousedown', handler);
  }, [isOpen, baseId]);
  
  // Escape key
  useEffect(() => {
    if (!isOpen) return;
    const handler = (e: KeyboardEvent) => {
      if (e.key === 'Escape') {
        setIsOpen(false);
        triggerRef.current?.focus();
      }
    };
    document.addEventListener('keydown', handler);
    return () => document.removeEventListener('keydown', handler);
  }, [isOpen]);
  
  const contextValue = useMemo<SelectContextValue>(() => ({
    value, setValue, isOpen, setIsOpen, baseId, triggerRef, registerItem, selectedLabel,
  }), [value, setValue, isOpen, baseId, registerItem, selectedLabel]);
  
  return (
    <SelectContext.Provider value={contextValue}>
      <div className="select">{children}</div>
    </SelectContext.Provider>
  );
}

export function SelectTrigger({ children }: { children: React.ReactNode }) {
  const { isOpen, setIsOpen, baseId, triggerRef } = useSelectContext('Select.Trigger');
  
  return (
    <button
      ref={triggerRef}
      role="combobox"
      aria-haspopup="listbox"
      aria-expanded={isOpen}
      aria-controls={`${baseId}-content`}
      className="select-trigger"
      onClick={() => setIsOpen(!isOpen)}
    >
      {children}
    </button>
  );
}

export function SelectValue({ placeholder }: { placeholder?: string }) {
  const { selectedLabel, value } = useSelectContext('Select.Value');
  return <span>{value ? selectedLabel : (placeholder ?? '')}</span>;
}

export function SelectContent({ children }: { children: React.ReactNode }) {
  const { isOpen, baseId } = useSelectContext('Select.Content');
  if (!isOpen) return null;
  
  return (
    <div
      id={`${baseId}-content`}
      role="listbox"
      className="select-content"
    >
      {children}
    </div>
  );
}

export function SelectItem({ 
  value: itemValue, 
  children 
}: { 
  value: string; 
  children: React.ReactNode;
}) {
  const { value, setValue, registerItem } = useSelectContext('Select.Item');
  const label = typeof children === 'string' ? children : itemValue;
  const isSelected = value === itemValue;
  
  useEffect(() => {
    registerItem(itemValue, label);
  }, [itemValue, label, registerItem]);
  
  return (
    <div
      role="option"
      aria-selected={isSelected}
      tabIndex={-1}
      className={`select-item ${isSelected ? 'selected' : ''}`}
      onClick={() => setValue(itemValue)}
    >
      {children}
    </div>
  );
}

Select.Trigger = SelectTrigger;
Select.Value = SelectValue;
Select.Content = SelectContent;
Select.Item = SelectItem;

// Usage
function FruitSelector() {
  const [selected, setSelected] = useState('apple');
  
  return (
    <Select value={selected} onChange={setSelected}>
      <Select.Trigger>
        <Select.Value placeholder="Choose a fruit..." />
        <span className="select-icon">▼</span>
      </Select.Trigger>
      
      <Select.Content>
        <Select.Item value="apple">Apple</Select.Item>
        <Select.Item value="banana">Banana</Select.Item>
        <Select.Item value="cherry">Cherry</Select.Item>
      </Select.Content>
    </Select>
  );
}
```

Bu — minimal Select. Production'da qo'shimcha:
- Portal (cross-ref [`28-portals.md`](28-portals.md))
- Position calculation (`@floating-ui/react`)
- Keyboard navigation (Arrow Up/Down, Enter)
- Search/filter
- Multi-select
- Group support

Library'lar (Radix UI Select, Headless UI Listbox) — barchasini implement qilgan.

</details>

---

## Real-World: Accordion Compound Component

### Nazariya

Accordion — collapsible sections list. Compound Components pattern bilan implement qilinadi.

API:

```tsx
<Accordion type="single" defaultValue="item-1">
  <Accordion.Item value="item-1">
    <Accordion.Trigger>What is React?</Accordion.Trigger>
    <Accordion.Content>React is a JavaScript library...</Accordion.Content>
  </Accordion.Item>
  
  <Accordion.Item value="item-2">
    <Accordion.Trigger>What are hooks?</Accordion.Trigger>
    <Accordion.Content>Hooks are functions...</Accordion.Content>
  </Accordion.Item>
</Accordion>
```

`type` prop:
- **`single`** — bir vaqtda faqat bitta item ochiq
- **`multiple`** — bir nechta item ochiq

Asosiy elementlar:

1. **`Accordion`** — root, state owner.
2. **`Accordion.Item`** — Individual collapsible item (state owner item-level).
3. **`Accordion.Trigger`** — Button to toggle open state.
4. **`Accordion.Content`** — Collapsible content.

State:
- `value: string | string[]` — open item value(s)
- `setValue: (value: string | string[]) => void`

`type='single'` — `value: string`, `type='multiple'` — `value: string[]`.

QANDAY ISHLAYDI:

1. `Accordion` Provider Context (open items state).
2. `Accordion.Item` o'zining `value` register qiladi va Item Context beradi.
3. `Accordion.Trigger` click — toggle item's open state.
4. `Accordion.Content` — open bo'lsa render.

Two-level Context:
- `AccordionContext` — root state (open items)
- `AccordionItemContext` — item-level state (current item value, isOpen)

NIMA UCHUN ikki Context:
- `Accordion.Item value="x"` — `value` register qilinishi kerak.
- `Accordion.Trigger` `Accordion.Item` ichidagi `value`'ni bilishi kerak (ammo trigger element-darajada `value` prop olmaydi).
- `AccordionItemContext` — Item children'ga `value`'ni provide qiladi.

```tsx
<Accordion type="single" defaultValue="item-1">
  <Accordion.Item value="item-1">           ← Item Context: value="item-1"
    <Accordion.Trigger>...</Accordion.Trigger>  ← reads item value from context
    <Accordion.Content>...</Accordion.Content>  ← reads item value
  </Accordion.Item>
</Accordion>
```

<details>
<summary><strong>Under the Hood</strong></summary>

Accordion ARIA structure:

```html
<div>
  <h3>
    <button 
      type="button" 
      aria-expanded="true" 
      aria-controls="content-1" 
      id="trigger-1"
    >
      What is React?
    </button>
  </h3>
  <div 
    id="content-1" 
    role="region" 
    aria-labelledby="trigger-1"
  >
    React is a JavaScript library...
  </div>
</div>
```

`role="region"` — ARIA landmark, screen reader navigation.

Animation — height 0 ↔ auto:

```css
.accordion-content {
  overflow: hidden;
  max-height: 0;
  transition: max-height 0.3s ease;
}

.accordion-content[data-state="open"] {
  max-height: 500px;  /* needs to be max possible value */
}
```

Modern CSS alternative — `interpolate-size: allow-keywords` (CSS-only feature, React'ga bog'liq emas, Chrome 129+ qo'llab-quvvatlaydi). `height: auto` ↔ `height: 0` transition'ga ruxsat beradi:

```css
:root {
  interpolate-size: allow-keywords;
}

.accordion-content {
  transition: height 0.3s;
  height: 0;
  overflow: hidden;
}

.accordion-content[data-state="open"] {
  height: auto;
}
```

Bu CSS feature — barcha browser'larda hali bor emas (Firefox, Safari hali support yo'q, 2026-yil holatiga ko'ra). Polyfill yo'q.

JavaScript-based animation — `framer-motion`, `react-spring` library'lari.

Accessibility keyboard:
- **Tab** — focus next trigger
- **Enter/Space** — toggle
- **Arrow Down/Up** (optional) — focus next/prev trigger

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

To'liq Accordion implementation:

```tsx
import React, { createContext, useContext, useState, useId, useMemo, useCallback } from 'react';

type AccordionType = 'single' | 'multiple';

interface AccordionContextValue {
  type: AccordionType;
  value: string | string[];
  toggleItem: (itemValue: string) => void;
  isItemOpen: (itemValue: string) => boolean;
  baseId: string;
}

const AccordionContext = createContext<AccordionContextValue | null>(null);

function useAccordionContext(componentName: string) {
  const ctx = useContext(AccordionContext);
  if (!ctx) throw new Error(`<${componentName}> must be inside <Accordion>`);
  return ctx;
}

interface AccordionItemContextValue {
  itemValue: string;
  isOpen: boolean;
  toggleItem: () => void;
  triggerId: string;
  contentId: string;
}

const AccordionItemContext = createContext<AccordionItemContextValue | null>(null);

function useAccordionItemContext(componentName: string) {
  const ctx = useContext(AccordionItemContext);
  if (!ctx) throw new Error(`<${componentName}> must be inside <Accordion.Item>`);
  return ctx;
}

interface AccordionPropsBase {
  children: React.ReactNode;
}

type AccordionProps = AccordionPropsBase & (
  | { type: 'single'; defaultValue?: string; value?: string; onChange?: (v: string) => void }
  | { type: 'multiple'; defaultValue?: string[]; value?: string[]; onChange?: (v: string[]) => void }
);

export function Accordion(props: AccordionProps) {
  const { type, children } = props;
  const baseId = useId();
  
  const [internalValue, setInternalValue] = useState<string | string[]>(
    props.defaultValue ?? (type === 'single' ? '' : [])
  );
  const isControlled = props.value !== undefined;
  const value = isControlled ? props.value! : internalValue;
  
  const setValue = useCallback((newValue: string | string[]) => {
    if (!isControlled) setInternalValue(newValue);
    if (type === 'single') (props as any).onChange?.(newValue as string);
    else (props as any).onChange?.(newValue as string[]);
  }, [isControlled, type, props]);
  
  const toggleItem = useCallback((itemValue: string) => {
    if (type === 'single') {
      const current = value as string;
      setValue(current === itemValue ? '' : itemValue);
    } else {
      const current = value as string[];
      setValue(
        current.includes(itemValue)
          ? current.filter(v => v !== itemValue)
          : [...current, itemValue]
      );
    }
  }, [type, value, setValue]);
  
  const isItemOpen = useCallback((itemValue: string) => {
    if (type === 'single') return value === itemValue;
    return (value as string[]).includes(itemValue);
  }, [type, value]);
  
  const contextValue = useMemo<AccordionContextValue>(() => ({
    type,
    value,
    toggleItem,
    isItemOpen,
    baseId,
  }), [type, value, toggleItem, isItemOpen, baseId]);
  
  return (
    <AccordionContext.Provider value={contextValue}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
}

export function AccordionItem({ 
  value: itemValue, 
  children 
}: { 
  value: string; 
  children: React.ReactNode;
}) {
  const { toggleItem, isItemOpen, baseId } = useAccordionContext('Accordion.Item');
  const isOpen = isItemOpen(itemValue);
  
  const itemContextValue = useMemo<AccordionItemContextValue>(() => ({
    itemValue,
    isOpen,
    toggleItem: () => toggleItem(itemValue),
    triggerId: `${baseId}-trigger-${itemValue}`,
    contentId: `${baseId}-content-${itemValue}`,
  }), [itemValue, isOpen, toggleItem, baseId]);
  
  return (
    <AccordionItemContext.Provider value={itemContextValue}>
      <div className="accordion-item" data-state={isOpen ? 'open' : 'closed'}>
        {children}
      </div>
    </AccordionItemContext.Provider>
  );
}

export function AccordionTrigger({ children }: { children: React.ReactNode }) {
  const { isOpen, toggleItem, triggerId, contentId } = useAccordionItemContext('Accordion.Trigger');
  
  return (
    <h3 className="accordion-header">
      <button
        id={triggerId}
        type="button"
        aria-expanded={isOpen}
        aria-controls={contentId}
        onClick={toggleItem}
        className="accordion-trigger"
      >
        {children}
        <span className="accordion-icon">{isOpen ? '−' : '+'}</span>
      </button>
    </h3>
  );
}

export function AccordionContent({ children }: { children: React.ReactNode }) {
  const { isOpen, triggerId, contentId } = useAccordionItemContext('Accordion.Content');
  
  return (
    <div
      id={contentId}
      role="region"
      aria-labelledby={triggerId}
      className="accordion-content"
      hidden={!isOpen}
    >
      {isOpen && <div className="accordion-content-inner">{children}</div>}
    </div>
  );
}

Accordion.Item = AccordionItem;
Accordion.Trigger = AccordionTrigger;
Accordion.Content = AccordionContent;

// Usage
function FAQSection() {
  return (
    <Accordion type="single" defaultValue="react">
      <Accordion.Item value="react">
        <Accordion.Trigger>What is React?</Accordion.Trigger>
        <Accordion.Content>
          React is a JavaScript library for building user interfaces.
        </Accordion.Content>
      </Accordion.Item>
      
      <Accordion.Item value="hooks">
        <Accordion.Trigger>What are hooks?</Accordion.Trigger>
        <Accordion.Content>
          Hooks are functions that let you "hook into" React state and lifecycle.
        </Accordion.Content>
      </Accordion.Item>
      
      <Accordion.Item value="vdom">
        <Accordion.Trigger>What is virtual DOM?</Accordion.Trigger>
        <Accordion.Content>
          Virtual DOM is a JavaScript representation of the actual DOM.
        </Accordion.Content>
      </Accordion.Item>
    </Accordion>
  );
}

// Multiple expansion
function FilterPanel() {
  return (
    <Accordion type="multiple" defaultValue={['category', 'price']}>
      <Accordion.Item value="category">
        <Accordion.Trigger>Category</Accordion.Trigger>
        <Accordion.Content>...</Accordion.Content>
      </Accordion.Item>
      
      <Accordion.Item value="price">
        <Accordion.Trigger>Price Range</Accordion.Trigger>
        <Accordion.Content>...</Accordion.Content>
      </Accordion.Item>
      
      <Accordion.Item value="rating">
        <Accordion.Trigger>Rating</Accordion.Trigger>
        <Accordion.Content>...</Accordion.Content>
      </Accordion.Item>
    </Accordion>
  );
}
```

Two-level Context — root va item — har biri o'z mas'uliyatiga ega. Trigger va Content item value'ni Item Context'dan oladi.

</details>

---

## Modern Compound vs Children API — Decision Guide

### Nazariya

Compound Components ikki implementation strategiyasi farq jadvali:

| Aspect | `cloneElement` | Context-based |
|--------|----------------|---------------|
| Direct children | ✅ | ✅ |
| Nested struktura | ❌ Recursion kerak | ✅ Native |
| Re-render scope | All children | Selected consumers |
| TypeScript | Manual prop typing | Auto inference |
| Custom hook integration | Yo'q | `useCustomContext` |
| DevTools | Cluttered (cloned elements) | Clean tree |
| Boilerplate | Past (1 wrapper) | O'rta (Context + Provider + hook) |
| Library API ergonomics | Yaxshi (caller drilling yo'q) | Yaxshi |
| SSR | OK | OK |
| React Compiler | Limited | Full optimization |

**Decision tree:**

```
Children state'ni share qilishi kerakmi?
├─ Ha
│   ├─ Children direct only? (`<Tabs>` ichida `<Tab>` to'g'ridan-to'g'ri)
│   │   ├─ Ha → cloneElement (legacy) yoki Context (modern)
│   │   └─ Yo'q (nested OK) → Context
│   └─ Children custom wrapper'lar bilan?
│       └─ Context (har chuqurlik ishlaydi)
└─ Yo'q (faqat structural)
    └─ No state — sodda Compound (Card.Header, Card.Body)
```

NIMA UCHUN Context default tanlov:
- Modern React idiomatic
- TypeScript inference yaxshi
- DevTools clean
- React Compiler optimization (1.0 stable, R19.1+ bilan 2025-aprel)
- Custom hook composition

NIMA UCHUN cloneElement hali kerak:
- Legacy codebase support
- Tooltip/Popover wrappers (single child)
- Library API'larda qisqa boilerplate

QANDAY ISHLAYDI: amaliy tanlov — **default Context**, lekin ba'zi cases'larda `cloneElement`:

```tsx
// ✅ Tooltip — single child, cloneElement OK
function Tooltip({ message, children }: { message: string; children: React.ReactElement }) {
  return React.cloneElement(children, { title: message });
}

// ✅ Tabs — multiple children, nested struktura, Context shart
function Tabs({ defaultValue, children }: TabsProps) {
  // ... Context provider
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

Performance comparison (5 children):

```
cloneElement approach:
  - Parent renders → Children.map → 5 cloneElement calls per render
  - Each clone creates new React Element object
  - Reconciler diffs new vs old element references → re-render

Context approach:
  - Parent renders → Provider re-renders if value changes
  - Consumers re-render only if Context value changes
  - useMemo on Context value → bailout if value stable
```

Memory cost:
- `cloneElement`: 5 new Element objects per render
- Context: 1 Provider Fiber + N consumer Fibers (re-render selectively)

Re-render frequency:
- `cloneElement`: All children re-render on parent state change
- Context: Only consumers (and their descendants) re-render

React Compiler (1.0 stable, R19.1+ bilan 2025-aprel; R17/18/19 mos):
- Auto-memoizes Context value (no manual `useMemo`)
- Inlines hook calls (less function overhead)
- `cloneElement` patterns harder to optimize (dynamic element creation)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Same component — two implementations:

```tsx
// === Implementation 1: cloneElement ===
function TabsCloned({ defaultValue, children }: { defaultValue: string; children: React.ReactNode }) {
  const [active, setActive] = useState(defaultValue);
  
  return (
    <div className="tabs">
      {React.Children.map(children, child => {
        if (!React.isValidElement(child)) return child;
        return React.cloneElement(child as React.ReactElement<TabClonedProps>, { active, setActive });
      })}
    </div>
  );
}

interface TabClonedProps {
  value: string;
  active?: string;
  setActive?: (v: string) => void;
  children: React.ReactNode;
}

function TabCloned({ value, active, setActive, children }: TabClonedProps) {
  return (
    <button
      className={active === value ? 'active' : ''}
      onClick={() => setActive?.(value)}
    >
      {children}
    </button>
  );
}

// Usage — direct children only
<TabsCloned defaultValue="profile">
  <TabCloned value="profile">Profile</TabCloned>
  <TabCloned value="settings">Settings</TabCloned>
</TabsCloned>

// ❌ Nested — TabCloned doesn't receive active/setActive
<TabsCloned defaultValue="profile">
  <div>
    <TabCloned value="profile">Profile</TabCloned>
  </div>
</TabsCloned>


// === Implementation 2: Context ===
const TabsCtx = createContext<{ active: string; setActive: (v: string) => void } | null>(null);

function TabsContext({ defaultValue, children }: { defaultValue: string; children: React.ReactNode }) {
  const [active, setActive] = useState(defaultValue);
  const value = useMemo(() => ({ active, setActive }), [active]);
  
  return (
    <TabsCtx.Provider value={value}>
      <div className="tabs">{children}</div>
    </TabsCtx.Provider>
  );
}

function TabContext({ value, children }: { value: string; children: React.ReactNode }) {
  const ctx = useContext(TabsCtx);
  if (!ctx) throw new Error('TabContext must be inside TabsContext');
  return (
    <button
      className={ctx.active === value ? 'active' : ''}
      onClick={() => ctx.setActive(value)}
    >
      {children}
    </button>
  );
}

// Usage — works at any depth
<TabsContext defaultValue="profile">
  <div>
    <div>
      <TabContext value="profile">Profile</TabContext>  {/* ✅ ishlaydi */}
    </div>
  </div>
</TabsContext>
```

Migration `cloneElement` → Context:

```tsx
// Before
function ProgressSteps({ children, currentStep }: { children: React.ReactNode; currentStep: number }) {
  return (
    <ol>
      {React.Children.map(children, (child, index) =>
        React.isValidElement(child)
          ? React.cloneElement(child, { isActive: index === currentStep, isPast: index < currentStep })
          : child
      )}
    </ol>
  );
}

// After (Context-based)
const StepContext = createContext<{ isActive: boolean; isPast: boolean } | null>(null);

function ProgressStepsContext({ children, currentStep }: { children: React.ReactNode; currentStep: number }) {
  return (
    <ol>
      {React.Children.map(children, (child, index) => {
        const value = { isActive: index === currentStep, isPast: index < currentStep };
        return (
          <StepContext.Provider key={index} value={value}>
            {child}
          </StepContext.Provider>
        );
      })}
    </ol>
  );
}

function Step({ children }: { children: React.ReactNode }) {
  const ctx = useContext(StepContext);
  if (!ctx) throw new Error('Step must be inside ProgressSteps');
  return (
    <li className={`${ctx.isActive ? 'active' : ''} ${ctx.isPast ? 'past' : ''}`}>
      {children}
    </li>
  );
}
```

Step component'ga prop drilling kerak emas, har chuqurlikda Context bilan ishlaydi.

</details>

---

## TypeScript Patterns for Compound Components

### Nazariya

Compound Components TypeScript bilan type-safe yozish — generic Context, strict consumer hook, attached subcomponent type'lari.

**Pattern 1: Strict Context value type**

```tsx
interface TabsContextValue {
  active: string;
  setActive: (value: string) => void;
}

const TabsContext = createContext<TabsContextValue | null>(null);

function useTabsContext(): TabsContextValue {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error('Component must be inside <Tabs>');
  return ctx;
}
```

`null` default + strict hook — production tavsiya pattern. Type narrow `if (!ctx)` keyin `ctx` non-null.

**Pattern 2: Static subcomponent attachment with types**

```tsx
function Tabs(props: TabsProps) { /* ... */ }
function Tab(props: TabProps) { /* ... */ }
function TabPanel(props: TabPanelProps) { /* ... */ }

// Type-safe attachment
type TabsType = typeof Tabs & {
  Tab: typeof Tab;
  Panel: typeof TabPanel;
};

const TabsExport = Tabs as TabsType;
TabsExport.Tab = Tab;
TabsExport.Panel = TabPanel;

export default TabsExport;
```

Static method'lar TypeScript'da — `typeof Tabs & { Tab: typeof Tab }` pattern.

Alternative — explicit interface:

```tsx
interface TabsCompound {
  (props: TabsProps): JSX.Element;
  Tab: typeof Tab;
  Panel: typeof TabPanel;
}

const Tabs: TabsCompound = ((props: TabsProps) => {
  // ...
}) as TabsCompound;

Tabs.Tab = Tab;
Tabs.Panel = TabPanel;
```

**Pattern 3: Generic Compound Component**

```tsx
interface SelectContextValue<T> {
  value: T;
  setValue: (value: T) => void;
}

function createSelectContext<T>() {
  return createContext<SelectContextValue<T> | null>(null);
}

// Usage
const StringSelectContext = createSelectContext<string>();
const NumberSelectContext = createSelectContext<number>();
```

Generic Context factory — re-usable for different value types.

**Pattern 4: Discriminated union for type variants**

```tsx
type AccordionProps = 
  | { type: 'single'; value?: string; onChange?: (v: string) => void; children: React.ReactNode }
  | { type: 'multiple'; value?: string[]; onChange?: (v: string[]) => void; children: React.ReactNode };

function Accordion(props: AccordionProps) {
  if (props.type === 'single') {
    // TypeScript narrows: props.value: string
  } else {
    // TypeScript narrows: props.value: string[]
  }
}
```

TypeScript discriminated union — `type` property based narrowing.

QANDAY ISHLAYDI: TypeScript inference bilan caller type-safe:

```tsx
<Tabs defaultValue="profile">
  <Tabs.Tab value="profile">Profile</Tabs.Tab>
  {/* ↑ TypeScript: value must be string */}
</Tabs>
```

`Tabs.Tab` props type'i `Tab` function signature'idan inferred.

<details>
<summary><strong>Under the Hood</strong></summary>

`createContext<T | null>` vs `createContext<T | undefined>`:

```tsx
// Variant 1: null default
const Ctx1 = createContext<TabsContextValue | null>(null);

function useCtx1() {
  const ctx = useContext(Ctx1);
  if (!ctx) throw new Error();  // narrows ctx to TabsContextValue
  return ctx;
}

// Variant 2: undefined default
const Ctx2 = createContext<TabsContextValue | undefined>(undefined);

function useCtx2() {
  const ctx = useContext(Ctx2);
  if (ctx === undefined) throw new Error();  // narrows
  return ctx;
}
```

Ikkalasi ham ishlaydi, `null` aksariyat library'larda standart.

`React.FC` vs `function` declaration:

```tsx
// ❌ Anti-pattern (cross-ref [`10-props.md`](10-props.md))
const Tabs: React.FC<TabsProps> = ({ children }) => { /* ... */ };

// ✅ Function declaration
function Tabs({ children }: TabsProps) { /* ... */ }
```

Static attachment `React.FC` bilan murakkab. Function declaration cleaner.

R19 ref oddiy prop — Compound Components ref forwarding:

```tsx
// R18 forwardRef
const Tab = React.forwardRef<HTMLButtonElement, TabProps>(({ value, children }, ref) => {
  // ...
  return <button ref={ref}>{children}</button>;
});

// R19 ref oddiy prop
function Tab({ value, children, ref }: TabProps & { ref?: React.Ref<HTMLButtonElement> }) {
  // ...
  return <button ref={ref}>{children}</button>;
}
```

R19 simplification — `forwardRef` wrapper kerak emas (cross-ref [`18-useref.md`](18-useref.md)).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

To'liq type-safe Tabs:

```tsx
import React, { createContext, useContext, useState, useMemo } from 'react';

// === Context types ===
interface TabsContextValue {
  active: string;
  setActive: (value: string) => void;
}

const TabsContext = createContext<TabsContextValue | null>(null);

function useTabsContext(componentName: string): TabsContextValue {
  const ctx = useContext(TabsContext);
  if (!ctx) {
    throw new Error(`<${componentName}> must be used inside <Tabs>`);
  }
  return ctx;
}

// === Component types ===
interface TabsProps {
  defaultValue: string;
  value?: string;
  onChange?: (value: string) => void;
  children: React.ReactNode;
}

interface TabsListProps extends React.HTMLAttributes<HTMLDivElement> {
  children: React.ReactNode;
}

interface TabProps extends Omit<React.ButtonHTMLAttributes<HTMLButtonElement>, 'value'> {
  value: string;
  children: React.ReactNode;
}

interface TabPanelProps {
  value: string;
  children: React.ReactNode;
}

// === Components ===
function TabsRoot({ defaultValue, value: controlled, onChange, children }: TabsProps) {
  const [internal, setInternal] = useState(defaultValue);
  const isControlled = controlled !== undefined;
  const active = isControlled ? controlled : internal;
  
  const setActive = (v: string) => {
    if (!isControlled) setInternal(v);
    onChange?.(v);
  };
  
  const ctxValue = useMemo<TabsContextValue>(() => ({ active, setActive }), [active]);
  
  return (
    <TabsContext.Provider value={ctxValue}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

function TabsList({ children, ...rest }: TabsListProps) {
  return <div role="tablist" {...rest}>{children}</div>;
}

function Tab({ value, children, ...rest }: TabProps) {
  const { active, setActive } = useTabsContext('Tab');
  return (
    <button
      role="tab"
      aria-selected={active === value}
      onClick={() => setActive(value)}
      {...rest}
    >
      {children}
    </button>
  );
}

function TabPanel({ value, children }: TabPanelProps) {
  const { active } = useTabsContext('TabPanel');
  if (active !== value) return null;
  return <div role="tabpanel">{children}</div>;
}

// === Type-safe attachment ===
type TabsCompound = typeof TabsRoot & {
  List: typeof TabsList;
  Tab: typeof Tab;
  Panel: typeof TabPanel;
};

const Tabs = TabsRoot as TabsCompound;
Tabs.List = TabsList;
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;

export default Tabs;

// === Usage with full type safety ===
import Tabs from './Tabs';

function App() {
  return (
    <Tabs defaultValue="profile">
      <Tabs.List aria-label="Settings">
        <Tabs.Tab value="profile">Profile</Tabs.Tab>
        <Tabs.Tab value="security">Security</Tabs.Tab>
      </Tabs.List>
      
      <Tabs.Panel value="profile">Profile content</Tabs.Panel>
      <Tabs.Panel value="security">Security content</Tabs.Panel>
    </Tabs>
  );
}
```

`Tabs.Tab` JSX TypeScript inference — `value: string` mandatory, `Tab.tabIndex`, `Tab.disabled` — `ButtonHTMLAttributes` orqali.

Interface-based Compound:

```tsx
interface AccordionRoot {
  (props: AccordionProps): JSX.Element;
  Item: (props: AccordionItemProps) => JSX.Element;
  Trigger: (props: AccordionTriggerProps) => JSX.Element;
  Content: (props: AccordionContentProps) => JSX.Element;
}

const Accordion: AccordionRoot = ((props: AccordionProps) => {
  // ...
}) as AccordionRoot;

Accordion.Item = AccordionItem;
Accordion.Trigger = AccordionTrigger;
Accordion.Content = AccordionContent;
```

Interface declaration — explicit type contract. `typeof` pattern lighter (re-uses function type).

</details>

---

## Accessibility — ARIA va Keyboard Navigation

### Nazariya

Compound Components — interactive UI primitive'lar (Tabs, Accordion, Select). **Accessibility** majburiy:

1. **ARIA roles** — semantic meaning for assistive technology.
2. **ARIA states** — `aria-expanded`, `aria-selected`, `aria-controls`.
3. **ARIA labels** — `aria-label`, `aria-labelledby`, `aria-describedby`.
4. **Keyboard navigation** — Tab, Arrow keys, Enter, Escape, Home/End.
5. **Focus management** — visible focus, focus return, focus trap.
6. **Screen reader announcements** — live regions, status updates.

WAI-ARIA Authoring Practices Guide (APG) — har komponent uchun ARIA pattern'lari belgilangan:

| Component | WAI-ARIA Pattern |
|-----------|------------------|
| Tabs | https://www.w3.org/WAI/ARIA/apg/patterns/tabs/ |
| Accordion | https://www.w3.org/WAI/ARIA/apg/patterns/accordion/ |
| Combobox (Select) | https://www.w3.org/WAI/ARIA/apg/patterns/combobox/ |
| Dialog | https://www.w3.org/WAI/ARIA/apg/patterns/dialog-modal/ |
| Menu | https://www.w3.org/WAI/ARIA/apg/patterns/menubar/ |

NIMA UCHUN: 15-20% foydalanuvchilar accessibility'ga muhtoj (vision impairment, motor impairment, cognitive). WCAG (Web Content Accessibility Guidelines) — legal requirement many countries.

QANDAY ISHLAYDI:

**`useId` (R18+) — SSR-safe unique IDs** (cross-ref [`22-concurrent-hooks.md`](22-concurrent-hooks.md)):

```tsx
function Accordion() {
  const baseId = useId();
  // baseId — SSR/client deterministic unique string (format React internal, e.g. ":r0:")
  
  return (
    <>
      <button id={`${baseId}-trigger-1`} aria-controls={`${baseId}-content-1`}>
        Trigger
      </button>
      <div id={`${baseId}-content-1`} aria-labelledby={`${baseId}-trigger-1`}>
        Content
      </div>
    </>
  );
}
```

**Keyboard navigation pattern** — Tab registry:

```tsx
const tabRefs = useRef<Map<string, HTMLButtonElement>>(new Map());

function registerTab(value: string, ref: HTMLButtonElement | null) {
  if (ref) tabRefs.current.set(value, ref);
  else tabRefs.current.delete(value);
}

function focusNextTab(currentValue: string) {
  const values = Array.from(tabRefs.current.keys());
  const idx = values.indexOf(currentValue);
  const nextValue = values[(idx + 1) % values.length];
  tabRefs.current.get(nextValue)?.focus();
}
```

**Focus trap** — Modal/Dialog'da focus ichida qoladi:

```tsx
useEffect(() => {
  if (!isOpen) return;
  
  const focusableSelector = 'button, a, input, [tabindex]:not([tabindex="-1"])';
  const focusableElements = modalRef.current?.querySelectorAll(focusableSelector);
  
  const handler = (e: KeyboardEvent) => {
    if (e.key !== 'Tab') return;
    
    const first = focusableElements?.[0] as HTMLElement;
    const last = focusableElements?.[focusableElements.length - 1] as HTMLElement;
    
    if (e.shiftKey && document.activeElement === first) {
      e.preventDefault();
      last?.focus();
    } else if (!e.shiftKey && document.activeElement === last) {
      e.preventDefault();
      first?.focus();
    }
  };
  
  document.addEventListener('keydown', handler);
  return () => document.removeEventListener('keydown', handler);
}, [isOpen]);
```

HTML `inert` attribute — modal'dan tashqari elementlar inert (focus + events ignored). R19'da React boolean prop sifatida proper render qiladi:

```tsx
<>
  <div inert={isOpen}>  {/* Modal ochiq bo'lsa, background inert */}
    <App />
  </div>
  <Modal />
</>
```

**Browser support:** Chrome 102+, Firefox 112+, Safari 15.5+ (HTML standard, React tomonidan kiritilmagan; cross-ref [`17-uselayouteffect.md`](17-uselayouteffect.md)).

<details>
<summary><strong>Under the Hood</strong></summary>

ARIA roles — semantic equivalents of HTML elements:

```html
<!-- Native HTML — implicit role -->
<button>Submit</button>  <!-- role="button" -->
<a href="...">Link</a>   <!-- role="link" -->

<!-- Custom — explicit role required -->
<div role="button" tabIndex={0} onClick={...}>Custom Button</div>
```

ARIA principle: **Don't use ARIA when native HTML works**. `<button>` afzal `<div role="button">`'dan.

ARIA states har gal sync bo'lishi shart:

```html
<button aria-expanded={isOpen} onClick={toggle}>
  Trigger
</button>
```

`aria-expanded` mismatch (DOM va ARIA) — screen reader yolg'on info.

Keyboard event'lar — `keydown` afzal `keypress` (deprecated):

```tsx
function handleKeyDown(e: KeyboardEvent) {
  switch (e.key) {
    case 'ArrowDown': /* ... */ break;
    case 'ArrowUp': /* ... */ break;
    case 'Enter':
    case ' ':       /* Space ham */ break;
    case 'Escape':  /* ... */ break;
    case 'Home':    /* ... */ break;
    case 'End':     /* ... */ break;
  }
}
```

`e.preventDefault()` — default browser behavior block (Tab dan tashqari, Tab focus management browser default qoladi keyboard-only navigation uchun).

Live regions — dynamic content announcements:

```html
<div aria-live="polite" aria-atomic="true">
  {message}
</div>
```

`aria-live="polite"` — screen reader idle bo'lsa announce, `aria-live="assertive"` — darrov.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Accessible Tabs (full keyboard support):

```tsx
function AccessibleTab({ value, children, disabled }: TabProps) {
  const { active, setActive, baseId, focusTab, getTabValues } = useTabsContext('Tab');
  const buttonRef = useRef<HTMLButtonElement>(null);
  const isActive = active === value;
  
  const handleKeyDown = (e: React.KeyboardEvent<HTMLButtonElement>) => {
    const values = getTabValues().filter(v => /* not disabled */ true);
    const currentIdx = values.indexOf(value);
    
    let targetIdx = currentIdx;
    
    switch (e.key) {
      case 'ArrowRight':
      case 'ArrowDown':
        targetIdx = (currentIdx + 1) % values.length;
        break;
      case 'ArrowLeft':
      case 'ArrowUp':
        targetIdx = (currentIdx - 1 + values.length) % values.length;
        break;
      case 'Home':
        targetIdx = 0;
        break;
      case 'End':
        targetIdx = values.length - 1;
        break;
      default:
        return;
    }
    
    e.preventDefault();
    const targetValue = values[targetIdx];
    setActive(targetValue);
    focusTab(targetValue);
  };
  
  return (
    <button
      ref={buttonRef}
      role="tab"
      id={`${baseId}-tab-${value}`}
      aria-selected={isActive}
      aria-controls={`${baseId}-panel-${value}`}
      tabIndex={isActive ? 0 : -1}
      disabled={disabled}
      onClick={() => setActive(value)}
      onKeyDown={handleKeyDown}
    >
      {children}
    </button>
  );
}
```

`tabIndex={isActive ? 0 : -1}` — **Roving tabindex** pattern. Faqat active tab focusable Tab key bilan, qolganlari Arrow keys orqali. Tab user faqat tablist'ga kirsa, keyin chiqsa.

Accessible Modal with focus trap:

```tsx
function AccessibleModal({ isOpen, onClose, title, children }: ModalProps) {
  const modalRef = useRef<HTMLDivElement>(null);
  const previousFocus = useRef<HTMLElement | null>(null);
  const titleId = useId();
  
  useEffect(() => {
    if (!isOpen) return;
    
    // Save focus
    previousFocus.current = document.activeElement as HTMLElement;
    
    // Focus first focusable in modal
    const focusables = modalRef.current?.querySelectorAll<HTMLElement>(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    focusables?.[0]?.focus();
    
    // Focus trap
    const handleTab = (e: KeyboardEvent) => {
      if (e.key !== 'Tab' || !focusables?.length) return;
      
      const first = focusables[0];
      const last = focusables[focusables.length - 1];
      
      if (e.shiftKey && document.activeElement === first) {
        e.preventDefault();
        last.focus();
      } else if (!e.shiftKey && document.activeElement === last) {
        e.preventDefault();
        first.focus();
      }
    };
    
    // Escape key
    const handleEscape = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onClose();
    };
    
    document.addEventListener('keydown', handleTab);
    document.addEventListener('keydown', handleEscape);
    
    // Body scroll lock
    document.body.style.overflow = 'hidden';
    
    return () => {
      document.removeEventListener('keydown', handleTab);
      document.removeEventListener('keydown', handleEscape);
      document.body.style.overflow = '';
      // Restore focus
      previousFocus.current?.focus();
    };
  }, [isOpen, onClose]);
  
  if (!isOpen) return null;
  
  return (
    <div
      className="modal-backdrop"
      role="dialog"
      aria-modal="true"
      aria-labelledby={titleId}
      onClick={onClose}
    >
      <div
        ref={modalRef}
        className="modal"
        onClick={(e) => e.stopPropagation()}
      >
        <h2 id={titleId}>{title}</h2>
        <button onClick={onClose} aria-label="Close">×</button>
        {children}
      </div>
    </div>
  );
}
```

Production-grade accessibility:
- Focus management (initial focus + return)
- Focus trap (Tab/Shift+Tab cycle)
- Escape key close
- Body scroll lock
- ARIA roles (`dialog`, `aria-modal`)
- ARIA labels (`aria-labelledby`)

</details>

---

## Edge Cases va Gotchas

### Gotcha 1: Static method vs named exports — Tree-shaking

```tsx
// ❌ Static methods — entire Tabs imported
import Tabs from './Tabs';
<Tabs.Tab />

// ✅ Named exports — tree-shakable
import { Tabs, TabsTab } from './Tabs';
<TabsTab />
```

Static method'lar — bundler entire `Tabs` exports (named imports yo'q `Tabs.Tab`). Tree-shaking limited. Library'lar (Radix UI) — named exports.

### Gotcha 2: Context value reference — har render new

```tsx
// ❌ Anti-pattern
function Tabs() {
  const [active, setActive] = useState('a');
  return (
    <TabsContext.Provider value={{ active, setActive }}>  {/* har render new object */}
      ...
    </TabsContext.Provider>
  );
}
```

Provider value har render yangi reference — barcha consumer'lar re-render. **Yechim**: `useMemo`:

```tsx
const value = useMemo(() => ({ active, setActive }), [active]);
```

`setActive` `useState` setter — bir xil reference (`useState` kafolati). `value` faqat `active` o'zgarganda yangilanadi.

React Compiler (1.0 stable, R19.1+ bilan 2025-aprel) — auto-memoize (manual `useMemo` kerak emas).

### Gotcha 3: `Children.map` key generation — collision

`Children.map` auto-key generates (`$.0`, `$.1`). Lekin user keys bilan combine bo'lganda silent collision:

```tsx
<Tabs>
  <Tab key="a" value="a">A</Tab>
  <Tab key="b" value="b">B</Tab>
</Tabs>

// Inside Tabs.Children.map:
// - child[0]: key = "a" → cloned with key "$.0/.$a"
// - child[1]: key = "b" → cloned with key "$.0/.$b"
```

Auto-prefix `$.0/.$` — collision'ni oldini oladi. Ammo manual cloneElement bilan key override qilinsa:

```tsx
React.cloneElement(child, { key: 'custom' });  // ❌ user key override
```

Original key yo'qoladi.

### Gotcha 4: Strict Mode 2x cycle — Compound Component setup

R18+ Strict Mode mount → cleanup → mount cycle. `registerTab` har mount'da chaqiriladi:

```tsx
function Tab({ value }: TabProps) {
  const { registerTab } = useTabsContext('Tab');
  const ref = useRef<HTMLButtonElement>(null);
  
  useEffect(() => {
    registerTab(value, ref.current);
    return () => registerTab(value, null);  // ✅ cleanup
  }, [value, registerTab]);
}
```

Cleanup — `registerTab(value, null)` yoki Map'dan delete. Aks holda Strict Mode 2x cycle'da map'da yolg'on entry qoladi.

### Gotcha 5: Context Provider tashqarida ishlatish — runtime error

```tsx
// ❌ Tab — Tabs tashqarida
function App() {
  return <Tab value="x">Tab</Tab>;  // useTabsContext throws
}
```

Strict consumer hook (`useTabsContext`) Provider'sini majbur qiladi. Test'larda mock Provider kerak:

```tsx
// Test setup
function renderWithTabs(ui: React.ReactElement) {
  return render(<Tabs defaultValue="x">{ui}</Tabs>);
}
```

---

## Common Mistakes

### ❌ Xato 1: Provider value `useMemo` yo'q

```tsx
// ❌ Har render yangi value object
function Tabs() {
  const [active, setActive] = useState('a');
  return <TabsContext.Provider value={{ active, setActive }}>...</TabsContext.Provider>;
}

// ✅ Memoized
function Tabs() {
  const [active, setActive] = useState('a');
  const value = useMemo(() => ({ active, setActive }), [active]);
  return <TabsContext.Provider value={value}>...</TabsContext.Provider>;
}
```

**Sabab:** Reference identity har render — barcha consumers re-render.

### ❌ Xato 2: `cloneElement` direct children only

```tsx
// ❌ Nested struktura — props inject yo'q
<Tabs>
  <div>
    <Tab value="a">A</Tab>  {/* ❌ Tab props olmaydi */}
  </div>
</Tabs>

// ✅ Direct children only
<Tabs>
  <Tab value="a">A</Tab>
  <Tab value="b">B</Tab>
</Tabs>

// ✅ Or use Context (works at any depth)
```

**Sabab:** `Children.map` faqat top-level children'ni iterate qiladi.

### ❌ Xato 3: ARIA attribute mismatch

```tsx
// ❌ aria-expanded sync emas
<button aria-expanded={true} onClick={toggle}>...</button>
// Static `true` — never updates

// ✅ Sync
<button aria-expanded={isOpen} onClick={toggle}>...</button>
```

**Sabab:** ARIA state DOM state bilan har gal sync bo'lishi shart.

### ❌ Xato 4: Provider'siz consumer

```tsx
// ❌ Consumer Provider tashqarida
function App() {
  return <Tab value="x">Tab</Tab>;  // useTabsContext throws
}

// ✅ Provider ichida
function App() {
  return (
    <Tabs defaultValue="x">
      <Tab value="x">Tab</Tab>
    </Tabs>
  );
}
```

**Sabab:** Strict consumer hook null ctx'da throw qiladi.

### ❌ Xato 5: Event listener cleanup yo'q

```tsx
// ❌ Cleanup yo'q
useEffect(() => {
  document.addEventListener('keydown', handler);
  // ❌ no return
}, []);

// ✅ Cleanup
useEffect(() => {
  document.addEventListener('keydown', handler);
  return () => document.removeEventListener('keydown', handler);
}, []);
```

**Sabab:** Memory leak + multiple listeners attached on Strict Mode 2x cycle.

---

## Amaliy Mashqlar

### Mashq 1: `Card` Static Compound Component (Oson)

`<Card>` + `<Card.Header>` + `<Card.Body>` + `<Card.Footer>` static compound component yarating. State sharing yo'q, faqat namespace organization.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import React from 'react';

interface CardProps extends React.HTMLAttributes<HTMLDivElement> {
  children: React.ReactNode;
}

function CardRoot({ children, className = '', ...rest }: CardProps) {
  return (
    <div className={`card ${className}`} {...rest}>
      {children}
    </div>
  );
}

function CardHeader({ children, className = '', ...rest }: CardProps) {
  return (
    <header className={`card-header ${className}`} {...rest}>
      {children}
    </header>
  );
}

function CardBody({ children, className = '', ...rest }: CardProps) {
  return (
    <div className={`card-body ${className}`} {...rest}>
      {children}
    </div>
  );
}

function CardFooter({ children, className = '', ...rest }: CardProps) {
  return (
    <footer className={`card-footer ${className}`} {...rest}>
      {children}
    </footer>
  );
}

type CardCompound = typeof CardRoot & {
  Header: typeof CardHeader;
  Body: typeof CardBody;
  Footer: typeof CardFooter;
};

const Card = CardRoot as CardCompound;
Card.Header = CardHeader;
Card.Body = CardBody;
Card.Footer = CardFooter;

export default Card;

// Usage
<Card>
  <Card.Header>
    <h2>Profile</h2>
  </Card.Header>
  <Card.Body>
    <p>User information here.</p>
  </Card.Body>
  <Card.Footer>
    <button>Edit</button>
  </Card.Footer>
</Card>
```

**Tushuntirish:**
- Har komponent independent (state sharing yo'q).
- Static method attachment (`Card.Header = CardHeader`).
- TypeScript `typeof CardRoot & {...}` — type-safe namespace.
- Standard HTML attributes spread (`...rest`).

</details>

### Mashq 2: `Toggle` Compound Component (Oson)

`<Toggle>` + `<Toggle.On>` + `<Toggle.Off>` + `<Toggle.Button>` Context-based compound. `<Toggle.On>` faqat on bo'lganda render, `<Toggle.Off>` off bo'lganda.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import React, { createContext, useContext, useState, useMemo, useCallback } from 'react';

interface ToggleContextValue {
  on: boolean;
  toggle: () => void;
  setOn: (value: boolean) => void;
}

const ToggleContext = createContext<ToggleContextValue | null>(null);

function useToggleContext(componentName: string): ToggleContextValue {
  const ctx = useContext(ToggleContext);
  if (!ctx) {
    throw new Error(`<${componentName}> must be used inside <Toggle>`);
  }
  return ctx;
}

interface ToggleProps {
  defaultOn?: boolean;
  onChange?: (on: boolean) => void;
  children: React.ReactNode;
}

function ToggleRoot({ defaultOn = false, onChange, children }: ToggleProps) {
  const [on, setOnState] = useState(defaultOn);
  
  const setOn = useCallback((value: boolean) => {
    setOnState(value);
    onChange?.(value);
  }, [onChange]);
  
  const toggle = useCallback(() => {
    setOnState(prev => {
      const next = !prev;
      onChange?.(next);
      return next;
    });
  }, [onChange]);
  
  const value = useMemo<ToggleContextValue>(
    () => ({ on, toggle, setOn }),
    [on, toggle, setOn]
  );
  
  return (
    <ToggleContext.Provider value={value}>
      {children}
    </ToggleContext.Provider>
  );
}

function ToggleOn({ children }: { children: React.ReactNode }) {
  const { on } = useToggleContext('Toggle.On');
  return on ? <>{children}</> : null;
}

function ToggleOff({ children }: { children: React.ReactNode }) {
  const { on } = useToggleContext('Toggle.Off');
  return !on ? <>{children}</> : null;
}

function ToggleButton(props: React.ButtonHTMLAttributes<HTMLButtonElement>) {
  const { toggle, on } = useToggleContext('Toggle.Button');
  return <button aria-pressed={on} {...props} onClick={toggle} />;
}

type ToggleCompound = typeof ToggleRoot & {
  On: typeof ToggleOn;
  Off: typeof ToggleOff;
  Button: typeof ToggleButton;
};

const Toggle = ToggleRoot as ToggleCompound;
Toggle.On = ToggleOn;
Toggle.Off = ToggleOff;
Toggle.Button = ToggleButton;

export default Toggle;

// Usage
function LightSwitch() {
  return (
    <Toggle defaultOn={false} onChange={(on) => console.log('Light:', on)}>
      <Toggle.On>
        <p>💡 The light is ON</p>
      </Toggle.On>
      <Toggle.Off>
        <p>🔌 The light is OFF</p>
      </Toggle.Off>
      <Toggle.Button>Toggle Light</Toggle.Button>
    </Toggle>
  );
}
```

**Tushuntirish:**
- Context-based state sharing.
- `Toggle.On`/`Toggle.Off` conditional render.
- `Toggle.Button` `aria-pressed` accessibility.
- Strict consumer hook (`useToggleContext` null check).
- `onChange` callback (controlled-like behavior).

</details>

### Mashq 3: `RadioGroup` Compound Component (O'rta)

`<RadioGroup>` + `<RadioGroup.Item>` Context-based. Selected state, keyboard navigation (Arrow keys), ARIA roles.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import React, { createContext, useContext, useState, useMemo, useCallback, useRef, useEffect } from 'react';

interface RadioGroupContextValue {
  value: string;
  setValue: (value: string) => void;
  registerItem: (value: string, ref: HTMLButtonElement | null) => void;
  focusItem: (value: string) => void;
  getItemValues: () => string[];
}

const RadioGroupContext = createContext<RadioGroupContextValue | null>(null);

function useRadioGroupContext(componentName: string) {
  const ctx = useContext(RadioGroupContext);
  if (!ctx) throw new Error(`<${componentName}> must be inside <RadioGroup>`);
  return ctx;
}

interface RadioGroupProps {
  defaultValue?: string;
  value?: string;
  onChange?: (value: string) => void;
  name: string;
  children: React.ReactNode;
}

function RadioGroupRoot({ defaultValue = '', value: controlled, onChange, name, children }: RadioGroupProps) {
  const [internal, setInternal] = useState(defaultValue);
  const isControlled = controlled !== undefined;
  const value = isControlled ? controlled : internal;
  
  const setValue = useCallback((newValue: string) => {
    if (!isControlled) setInternal(newValue);
    onChange?.(newValue);
  }, [isControlled, onChange]);
  
  const itemRefs = useRef<Map<string, HTMLButtonElement>>(new Map());
  
  const registerItem = useCallback((itemValue: string, ref: HTMLButtonElement | null) => {
    if (ref) itemRefs.current.set(itemValue, ref);
    else itemRefs.current.delete(itemValue);
  }, []);
  
  const focusItem = useCallback((itemValue: string) => {
    itemRefs.current.get(itemValue)?.focus();
  }, []);
  
  const getItemValues = useCallback(() => Array.from(itemRefs.current.keys()), []);
  
  const ctxValue = useMemo<RadioGroupContextValue>(() => ({
    value, setValue, registerItem, focusItem, getItemValues,
  }), [value, setValue, registerItem, focusItem, getItemValues]);
  
  return (
    <RadioGroupContext.Provider value={ctxValue}>
      <div role="radiogroup" data-name={name}>
        {children}
      </div>
    </RadioGroupContext.Provider>
  );
}

interface RadioItemProps {
  value: string;
  children: React.ReactNode;
  disabled?: boolean;
}

function RadioItem({ value: itemValue, children, disabled }: RadioItemProps) {
  const { value, setValue, registerItem, focusItem, getItemValues } = useRadioGroupContext('RadioGroup.Item');
  const buttonRef = useRef<HTMLButtonElement>(null);
  const isSelected = value === itemValue;
  
  useEffect(() => {
    registerItem(itemValue, buttonRef.current);
    return () => registerItem(itemValue, null);
  }, [itemValue, registerItem]);
  
  const handleKeyDown = (e: React.KeyboardEvent<HTMLButtonElement>) => {
    const values = getItemValues();
    const currentIdx = values.indexOf(itemValue);
    
    let targetIdx = currentIdx;
    
    switch (e.key) {
      case 'ArrowRight':
      case 'ArrowDown':
        targetIdx = (currentIdx + 1) % values.length;
        break;
      case 'ArrowLeft':
      case 'ArrowUp':
        targetIdx = (currentIdx - 1 + values.length) % values.length;
        break;
      default:
        return;
    }
    
    e.preventDefault();
    const targetValue = values[targetIdx];
    setValue(targetValue);
    focusItem(targetValue);
  };
  
  return (
    <button
      ref={buttonRef}
      role="radio"
      aria-checked={isSelected}
      tabIndex={isSelected ? 0 : -1}
      disabled={disabled}
      className={`radio-item ${isSelected ? 'selected' : ''}`}
      onClick={() => setValue(itemValue)}
      onKeyDown={handleKeyDown}
    >
      <span className="radio-indicator" aria-hidden="true">
        {isSelected ? '●' : '○'}
      </span>
      <span>{children}</span>
    </button>
  );
}

type RadioGroupCompound = typeof RadioGroupRoot & {
  Item: typeof RadioItem;
};

const RadioGroup = RadioGroupRoot as RadioGroupCompound;
RadioGroup.Item = RadioItem;

export default RadioGroup;

// Usage
function PaymentForm() {
  const [method, setMethod] = useState('card');
  
  return (
    <RadioGroup name="payment" value={method} onChange={setMethod}>
      <RadioGroup.Item value="card">Credit Card</RadioGroup.Item>
      <RadioGroup.Item value="paypal">PayPal</RadioGroup.Item>
      <RadioGroup.Item value="bank">Bank Transfer</RadioGroup.Item>
    </RadioGroup>
  );
}
```

**Tushuntirish:**
- Roving tabindex (`tabIndex={isSelected ? 0 : -1}`).
- Keyboard navigation (Arrow keys cycle).
- ARIA roles (`radiogroup`, `radio`, `aria-checked`).
- Item registry (Map<value, ref>).
- Controlled/Uncontrolled support.

</details>

### Mashq 4: `cloneElement` → Context migration (O'rta)

Berilgan `Stepper` `cloneElement` implementation'ni Context-based variant'ga refactor qiling. Backward compatibility — caller API o'zgartirmaydi.

```tsx
// Original cloneElement implementation
function Stepper({ children, currentStep }: { children: React.ReactNode; currentStep: number }) {
  return (
    <ol className="stepper">
      {React.Children.map(children, (child, index) => {
        if (!React.isValidElement(child)) return child;
        return React.cloneElement(child, {
          stepNumber: index + 1,
          isActive: index === currentStep,
          isCompleted: index < currentStep,
        });
      })}
    </ol>
  );
}

interface StepProps {
  stepNumber?: number;
  isActive?: boolean;
  isCompleted?: boolean;
  children: React.ReactNode;
}

function Step({ stepNumber, isActive, isCompleted, children }: StepProps) {
  return (
    <li className={`step ${isActive ? 'active' : ''} ${isCompleted ? 'completed' : ''}`}>
      <span className="step-number">{stepNumber}</span>
      <span className="step-label">{children}</span>
    </li>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
import React, { createContext, useContext, useMemo } from 'react';

// === Context-based implementation ===
interface StepperContextValue {
  currentStep: number;
}

const StepperContext = createContext<StepperContextValue | null>(null);

interface StepContextValue {
  stepIndex: number;
  isActive: boolean;
  isCompleted: boolean;
}

const StepContext = createContext<StepContextValue | null>(null);

function useStepperContext() {
  const ctx = useContext(StepperContext);
  if (!ctx) throw new Error('Step must be inside <Stepper>');
  return ctx;
}

function useStepContext() {
  const ctx = useContext(StepContext);
  if (!ctx) throw new Error('Step content must be inside <Step>');
  return ctx;
}

interface StepperProps {
  currentStep: number;
  children: React.ReactNode;
}

export function Stepper({ currentStep, children }: StepperProps) {
  const value = useMemo<StepperContextValue>(() => ({ currentStep }), [currentStep]);
  
  // Wrap each Step with index-based StepContext.Provider
  const wrappedChildren = React.Children.map(children, (child, index) => {
    if (!React.isValidElement(child)) return child;
    
    const stepValue: StepContextValue = {
      stepIndex: index,
      isActive: index === currentStep,
      isCompleted: index < currentStep,
    };
    
    return (
      <StepContext.Provider value={stepValue}>
        {child}
      </StepContext.Provider>
    );
  });
  
  return (
    <StepperContext.Provider value={value}>
      <ol className="stepper">{wrappedChildren}</ol>
    </StepperContext.Provider>
  );
}

interface StepProps {
  children: React.ReactNode;
}

export function Step({ children }: StepProps) {
  const { stepIndex, isActive, isCompleted } = useStepContext();
  
  return (
    <li className={`step ${isActive ? 'active' : ''} ${isCompleted ? 'completed' : ''}`}>
      <span className="step-number">{stepIndex + 1}</span>
      <span className="step-label">{children}</span>
    </li>
  );
}

// Usage — caller API o'zgarmadi
<Stepper currentStep={1}>
  <Step>Personal Info</Step>
  <Step>Address</Step>
  <Step>Confirmation</Step>
</Stepper>
```

**Tushuntirish:**
- `cloneElement` → Context (StepContext per Step).
- `Step` props (`stepNumber`, `isActive`, `isCompleted`) → Context value.
- `Step` component children'lari Context'ga kira oladi (nested struktura ishlaydi).
- Caller API saqlangan — backward compatible.

**Trade-off:**
- More boilerplate (2 Contexts).
- Better composition (nested Step children Context'ga kira oladi).
- DevTools cleaner (no cloneElement).

</details>

### Mashq 5: `Disclosure` Compound Component (Qiyin)

`<Disclosure>` (collapsible section) compound component yarating. Asosiy: `<Disclosure>` + `<Disclosure.Trigger>` + `<Disclosure.Panel>`. Animation `framer-motion` yo'q — pure CSS height transition. Accessibility (ARIA, keyboard).

<details>
<summary><strong>Javob</strong></summary>

```tsx
import React, { createContext, useContext, useState, useMemo, useCallback, useId, useRef, useEffect } from 'react';

interface DisclosureContextValue {
  isOpen: boolean;
  toggle: () => void;
  triggerId: string;
  panelId: string;
}

const DisclosureContext = createContext<DisclosureContextValue | null>(null);

function useDisclosureContext(componentName: string) {
  const ctx = useContext(DisclosureContext);
  if (!ctx) throw new Error(`<${componentName}> must be inside <Disclosure>`);
  return ctx;
}

interface DisclosureProps {
  defaultOpen?: boolean;
  open?: boolean;
  onOpenChange?: (open: boolean) => void;
  children: React.ReactNode;
}

function DisclosureRoot({ defaultOpen = false, open: controlled, onOpenChange, children }: DisclosureProps) {
  const [internalOpen, setInternalOpen] = useState(defaultOpen);
  const isControlled = controlled !== undefined;
  const isOpen = isControlled ? controlled : internalOpen;
  
  const toggle = useCallback(() => {
    const newValue = !isOpen;
    if (!isControlled) setInternalOpen(newValue);
    onOpenChange?.(newValue);
  }, [isControlled, isOpen, onOpenChange]);
  
  const baseId = useId();
  const triggerId = `${baseId}-trigger`;
  const panelId = `${baseId}-panel`;
  
  const value = useMemo<DisclosureContextValue>(
    () => ({ isOpen, toggle, triggerId, panelId }),
    [isOpen, toggle, triggerId, panelId]
  );
  
  return (
    <DisclosureContext.Provider value={value}>
      <div className="disclosure" data-state={isOpen ? 'open' : 'closed'}>
        {children}
      </div>
    </DisclosureContext.Provider>
  );
}

function DisclosureTrigger({ children, ...rest }: React.ButtonHTMLAttributes<HTMLButtonElement>) {
  const { isOpen, toggle, triggerId, panelId } = useDisclosureContext('Disclosure.Trigger');
  
  return (
    <button
      id={triggerId}
      type="button"
      aria-expanded={isOpen}
      aria-controls={panelId}
      onClick={toggle}
      {...rest}
    >
      {children}
    </button>
  );
}

function DisclosurePanel({ children }: { children: React.ReactNode }) {
  const { isOpen, triggerId, panelId } = useDisclosureContext('Disclosure.Panel');
  const contentRef = useRef<HTMLDivElement>(null);
  const [height, setHeight] = useState<number | 'auto'>(isOpen ? 'auto' : 0);
  
  useEffect(() => {
    if (!contentRef.current) return;
    
    if (isOpen) {
      // Open — measure content height
      const measuredHeight = contentRef.current.scrollHeight;
      setHeight(measuredHeight);
      
      // After transition, set to auto for responsive content
      const timer = setTimeout(() => setHeight('auto'), 300);
      return () => clearTimeout(timer);
    } else {
      // Close — measure current height first, then collapse
      const currentHeight = contentRef.current.getBoundingClientRect().height;
      setHeight(currentHeight);
      
      // Force reflow
      requestAnimationFrame(() => setHeight(0));
    }
  }, [isOpen]);
  
  return (
    <div
      id={panelId}
      role="region"
      aria-labelledby={triggerId}
      style={{
        height: typeof height === 'number' ? `${height}px` : height,
        overflow: 'hidden',
        transition: 'height 0.3s ease',
      }}
    >
      <div ref={contentRef} className="disclosure-panel-inner">
        {children}
      </div>
    </div>
  );
}

type DisclosureCompound = typeof DisclosureRoot & {
  Trigger: typeof DisclosureTrigger;
  Panel: typeof DisclosurePanel;
};

const Disclosure = DisclosureRoot as DisclosureCompound;
Disclosure.Trigger = DisclosureTrigger;
Disclosure.Panel = DisclosurePanel;

export default Disclosure;

// Usage
function FAQ() {
  return (
    <div>
      <Disclosure>
        <Disclosure.Trigger>What is React?</Disclosure.Trigger>
        <Disclosure.Panel>
          <p>React is a JavaScript library for building user interfaces.</p>
        </Disclosure.Panel>
      </Disclosure>
      
      <Disclosure defaultOpen>
        <Disclosure.Trigger>What are hooks?</Disclosure.Trigger>
        <Disclosure.Panel>
          <p>Hooks are functions that let you use state and lifecycle in function components.</p>
        </Disclosure.Panel>
      </Disclosure>
      
      <Disclosure>
        <Disclosure.Trigger>Custom controlled</Disclosure.Trigger>
        <Disclosure.Panel>
          <p>Controlled by parent state.</p>
        </Disclosure.Panel>
      </Disclosure>
    </div>
  );
}
```

**Tushuntirish:**
- Context-based state.
- Controlled/Uncontrolled.
- ARIA (`aria-expanded`, `aria-controls`, `role="region"`, `aria-labelledby`).
- `useId` SSR-safe.
- Height animation:
  - Open: 0 → measured height → auto (responsive).
  - Close: auto → current height → 0 (transition).
- `requestAnimationFrame` — force reflow before transition.

**Production caveats:**
- Animation library (`framer-motion`) yaxshi (spring physics, interrupt handling).
- `interpolate-size: allow-keywords` CSS — native height auto animation (modern browsers).
- Multiple disclosure'lar bir Accordion sifatida birlashtirilishi mumkin.

</details>

---

## Xulosa

Compound Components — UI library design'ning fundamental pattern'i. Hozirgi kunda ko'pchilik UI library (Radix UI, Headless UI, Material UI, shadcn/ui) shu pattern'da qurilgan. Asosiy fikrlar:

- **Compound Components Nima** — bir nechta komponent **bitta logical unit** sifatida ishlaydi. State va behavior parent boshqaradi, children consume qiladi. HTML klassik analog: `<select>` + `<option>`. **Maqsad** flexible, declarative API + implicit state sharing + encapsulation.
- **Ikki Implementation Strategy** — `React.Children` API + `cloneElement` (legacy, direct children only) yoki Context-based (modern, har chuqurlikda nested struktura).
- **`React.Children` API** — `Children.map` (auto-key, skip null), `Children.toArray` (filter + flatten), `Children.count` (node count), `Children.only` (single child validation), `Children.forEach` (no return). Native `arr.map` va `Children.map` farq — `null`/`false` handling + auto-key.
- **`cloneElement`** — Element kloni + qo'shimcha props inject. Shallow merge (last-wins). `key`/`ref` saqlanadi yoki override. **Cheklov'lar**: faqat React Element, direct children, props collision (event handler manual chain), ref re-attach.
- **`Children.only` va `Children.count`** — validation va layout decisions. `Children.only` single child shart (Tooltip). `Children.count` `null`/`boolean`/`undefined` skip qiladi.
- **Context-Based (Modern)** — Provider value + `useContext`. **Avantajlar:** har chuqurlikda nested, TypeScript inference, React Compiler optimization (1.0 stable, R19.1+ bilan 2025-aprel), custom hook integration, DevTools clean. Strict consumer hook (`useTabsContext` Provider'sini majbur qiladi).
- **Real-World Tabs** — `Tabs` + `Tabs.List` + `Tabs.Tab` + `Tabs.Panel`. Controlled/Uncontrolled, ARIA roles (`tablist`, `tab`, `tabpanel`), keyboard navigation (Arrow keys, Home/End), Roving tabindex pattern, `useId` SSR-safe IDs, Tab registry (focus next/prev).
- **Real-World Select** — `Select` + `Trigger` + `Content` + `Item`. Dropdown ochish/yopish, click outside (`useOnClickOutside` cross-ref 24), Escape key close, Portal (cross-ref 28), position calculation, ARIA combobox/listbox/option.
- **Real-World Accordion** — `Accordion` + `Item` + `Trigger` + `Content`. Two-level Context (root + item-level), single/multiple expansion (discriminated union props), CSS height animation, ARIA region.
- **Modern Compound vs Children API Decision Guide** — direct children + nested struktura jadval. Default tanlov **Context** (modern idiomatic, React Compiler bilan optimal). `cloneElement` legacy support, single-child wrappers (Tooltip).
- **TypeScript Patterns** — strict Context value type (`null` default + strict hook), static method attachment (`typeof X & { Y: typeof Y }`), generic Context factory, discriminated union (Accordion `single`/`multiple`), Compound interface explicit declaration.
- **Accessibility** — WAI-ARIA APG patterns (Tabs, Accordion, Combobox, Dialog), `useId` (R18+) SSR-safe IDs, keyboard navigation (Arrow keys, Tab, Enter, Escape, Home/End), Roving tabindex, focus trap (Modal), focus return on close, ARIA states (`aria-expanded`, `aria-selected`, `aria-checked`), live regions, HTML `inert` attribute (R19+ React boolean prop support).
- **Edge Cases** — Static method tree-shaking (named exports afzal library'larda), Provider value `useMemo` (har render new object), `Children.map` auto-key collision, Strict Mode 2x cycle setup cleanup, Provider'siz consumer runtime error.

Versiya evolyutsiyasi:
- Pre-R16.3: `React.Children` + `cloneElement` dominant
- R16.3+: Stable `createContext` API, Context-based variant boshlandi
- R16.8+: `useContext` hook — modern Compound default
- R19+: `<Context value>` shorthand, `use(context)` conditional, ref oddiy prop, Compiler auto-memoization

Cross-references:

- [`11-composition.md`](11-composition.md) — Slot pattern, polymorphic component
- [`12-state-and-usestate.md`](12-state-and-usestate.md) — `useState` controlled/uncontrolled
- [`14-lifting-and-controlled.md`](14-lifting-and-controlled.md) — Controlled vs Uncontrolled pattern
- [`18-useref.md`](18-useref.md) — `forwardRef` R18 → R19 ref oddiy prop
- [`19-usecontext.md`](19-usecontext.md) — Context Provider/Consumer base, splitting Contexts
- [`22-concurrent-hooks.md`](22-concurrent-hooks.md) — `useId` SSR-safe IDs
- [`23-r19-hooks.md`](23-r19-hooks.md) — `use(context)` conditional reading R19
- [`24-custom-hooks.md`](24-custom-hooks.md) — `useOnClickOutside`, `useEventListener`
- [`28-portals.md`](28-portals.md) — Portal Modal/Dropdown
- [`31-react-compiler.md`](31-react-compiler.md) — Auto-memoization

---

**Keyingi bo'lim:** [27-error-boundaries.md](27-error-boundaries.md) — Error Boundaries: class component minimal primer (faqat error boundary uchun zarur lifecycle methodlar `componentDidMount`/`componentDidUpdate`/`componentWillUnmount` hooks bilan ekvivalent jadval), `getDerivedStateFromError` va `componentDidCatch` (hooks bilan almashtirib bo'lmaydi), error catching scope (render/lifecycle/constructor — NOT event handlers/async/SSR), placement strategy granular vs global, error recovery `key` prop bilan boundary reset, `react-error-boundary` library mention, R19 root callbacks (`onCaughtError`/`onUncaughtError`/`onRecoverableError`), versiya evolyutsiyasi callout (Class lifecycle → Hooks, `UNSAFE_*` deprecated lifecycle methods).
