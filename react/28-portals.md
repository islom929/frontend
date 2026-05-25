# Bo'lim 28: Portals

> Portal — React'ning komponent'ni DOM tree'ning **boshqa joyiga** render qiluvchi mexanizmi. `createPortal(children, container)` API komponent JSX joyida yoziladi, lekin natija boshqa DOM element ichida render qilinadi (odatda `document.body`). Bu Modal, Tooltip, Dropdown, Notification kabi UI primitive'lar uchun fundamental — chunki ular parent'lardagi `overflow: hidden`, `z-index`, va `transform` cheklov'laridan qutulishi kerak. Lekin **React tree'da Portal ichidagi element'lar parent komponent ichida qoladi** — Context, event bubbling, va Error Boundary normal ishlaydi. Bu **fundamental farq** Portal'larning eng muhim xususiyati.

---

## Mundarija

- [Portals Nima va Nima Uchun](#portals-nima-va-nima-uchun)
- [`createPortal` API](#createportal-api)
- [DOM Tree vs React Tree — Fundamental Farq](#dom-tree-vs-react-tree--fundamental-farq)
- [Event Bubbling React Tree Bo'ylab](#event-bubbling-react-tree-boylab)
- [Real-World: Modal Implementation](#real-world-modal-implementation)
- [Real-World: Tooltip Implementation](#real-world-tooltip-implementation)
- [Real-World: Dropdown va Popover](#real-world-dropdown-va-popover)
- [Real-World: Toast / Notification System](#real-world-toast--notification-system)
- [z-index va Stacking Contexts](#z-index-va-stacking-contexts)
- [Focus Management va Focus Trap](#focus-management-va-focus-trap)
- [SSR Considerations](#ssr-considerations)
- [Portal Cleanup va Lifecycle](#portal-cleanup-va-lifecycle)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Portals Nima va Nima Uchun

### Nazariya

**Portal** — React komponentni DOM daraxtining **boshqa joyiga** render qilish mexanizmi. JSX'da komponent o'z parent'i ichida yoziladi (logical placement), lekin DOM'da boshqa joyda render qilinadi (visual placement).

```tsx
import { createPortal } from 'react-dom';

function Modal({ isOpen, children }: { isOpen: boolean; children: React.ReactNode }) {
  if (!isOpen) return null;
  
  return createPortal(
    <div className="modal-backdrop">
      <div className="modal">{children}</div>
    </div>,
    document.body  // DOM target — body'ga render
  );
}

// Usage
function App() {
  return (
    <div className="app">
      <Header />
      <Sidebar>
        <Modal isOpen={true}>
          <p>Modal content</p>
        </Modal>
      </Sidebar>
    </div>
  );
}
```

JSX'da Modal Sidebar ichida yozilgan (logical), lekin DOM'da `<body>`'da render qilingan (visual). React tree'da Modal Sidebar parent'iga bog'langan (Context, events).

NIMA UCHUN Portal kerak:

1. **`overflow: hidden` parent'lardan qutulish** — Sidebar'da `overflow: hidden` bo'lsa, ichidagi Modal ko'rinmaydi. Portal'da `<body>`'ga render qilsangiz — Sidebar overflow ahamiyatsiz.

2. **`z-index` stacking contexts** — Sidebar'da `z-index: 1` bo'lsa, ichidagi Modal `z-index: 9999` qo'ysangiz ham parent stacking context'da qoladi. `<body>`'da render qilsangiz — top-level stacking.

3. **`transform`/`filter` parent'lar** — CSS `transform` parent stacking context yaratadi va ichidagi `position: fixed` ham parent'ga "bog'lanadi" (not viewport). Portal `<body>`'ga render qilsa — viewport-relative.

4. **Visual placement vs Logical placement** — Modal/Tooltip/Toast UI'da top-level joylashishi kerak (visual), lekin logically parent komponent'ga aloqador (state, events).

QANDAY ISHLAYDI: React JSX render paytida `createPortal` chaqirilganda:

1. React Element create qilinadi (bu `$$typeof === REACT_PORTAL_TYPE` belgisi bilan).
2. React Reconciler bu element'ni alohida treat qiladi.
3. Render paytida — children'lar **container** element'iga DOM API orqali insert qilinadi (`appendChild`).
4. React tree'da Portal element parent komponent'i bilan bog'langan (events bubble, Context inherits).
5. Unmount paytida — children'lar container'dan `removeChild` qilinadi.

```
Render:
  <App>
    <Sidebar>
      <Modal>          ← React tree'da Sidebar > Modal
                       ← DOM: body > Modal (Portal jumps to body)

DOM tree:
  <body>
    <div id="app">...</div>
    <div class="modal-backdrop">  ← Portal'dan
      <div class="modal">...</div>
    </div>
  </body>
```

> **Versiya evolyutsiyasi (Portals):**
> - **Pre-R16:** Rasmiy `createPortal` mavjud emas. Workaround sifatida `ReactDOM.unstable_renderSubtreeIntoContainer` (experimental API) yoki `componentDidMount` ichida manual DOM manipulation ishlatilardi.
> - **R16 (2017):** `createPortal(children, container)` API kiritildi. DOM tree va React tree ajratish standartlashdi. Error Boundary (R16.0) va Suspense (R16.6) Portal children'larini React tree bo'ylab ushlaydi (DOM joyi ahamiyatsiz — birinchi versiyadan beri).
> - **R18 (2022):** Portal Concurrent rendering bilan ishlaydi — `useTransition`, `useDeferredValue`, automatic batching, va concurrent Suspense Portal children'larida ham normal ishlaydi. Selective hydration boundary'lar Portal'lar atrofida ham.
> - **R19 (2024):** Portal API o'zgarmagan. Lekin `<title>`/`<meta>` document metadata Portal alternative R19 native (cross-ref [`37-react-19-document-apis.md`](37-react-19-document-apis.md)). HTML `inert` boolean prop sifatida proper render qilinadi (pre-R19 string serialization warning bergan).

<details>
<summary><strong>Under the Hood</strong></summary>

`createPortal` source code (simplified):

```javascript
// react/src/ReactPortal.js
function createPortal(children, containerInfo, key) {
  return {
    $$typeof: REACT_PORTAL_TYPE,
    key: key == null ? null : '' + key,
    children,
    containerInfo,
    implementation: null,
  };
}
```

`REACT_PORTAL_TYPE` — `Symbol.for('react.portal')`. React Reconciler'ga signal — "bu element boshqa container'ga render qilinadi".

Reconciler Portal handling:

```javascript
// react-reconciler/src/ReactFiberBeginWork.js (simplified)
function beginWork(workInProgress) {
  switch (workInProgress.tag) {
    case HostPortal: {
      // Portal Fiber'i
      pushHostContainer(workInProgress, workInProgress.stateNode.containerInfo);
      reconcileChildren(workInProgress, workInProgress.pendingProps);
      return workInProgress.child;
    }
    // ... other cases
  }
}
```

Portal Fiber'i `tag: HostPortal`. Render paytida `containerInfo` (DOM element) push qilinadi va children shu container'ga commit qilinadi.

Commit Phase:

```javascript
// react-dom-bindings/src/client/ReactDOMHostConfig.js (simplified)
function commitPortal(portalFiber) {
  const container = portalFiber.stateNode.containerInfo;
  
  portalFiber.children.forEach(child => {
    container.appendChild(child.stateNode);
  });
}
```

Portal children DOM'ga `container` ichiga append qilinadi (default `document.body` yoki user-provided element).

Performance impact:
- Portal Fiber'i bir qo'shimcha Fiber node sifatida tree'da turadi (har Fiber'ning struktura overhead'i: tag, stateNode, child/sibling/return pointer'lari, alternate, va h.k.).
- Render paytida bir qo'shimcha traversal qadami — boshqa Fiber tag'lariga (HostComponent, FunctionComponent) nisbatan farq sezilarli emas.
- DOM commit'da `appendChild`/`removeChild` — bir element ko'chirish, native DOM operation.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Sodda Portal — Modal:

```tsx
import React from 'react';
import { createPortal } from 'react-dom';

function Modal({ isOpen, onClose, children }: { 
  isOpen: boolean; 
  onClose: () => void; 
  children: React.ReactNode;
}) {
  if (!isOpen) return null;
  
  return createPortal(
    <div className="modal-backdrop" onClick={onClose}>
      <div className="modal" onClick={(e) => e.stopPropagation()}>
        {children}
      </div>
    </div>,
    document.body
  );
}

// Usage — Modal Sidebar ichida (overflow:hidden parent'siz)
function Sidebar() {
  const [isModalOpen, setIsModalOpen] = useState(false);
  
  return (
    <aside style={{ overflow: 'hidden', maxHeight: '500px' }}>
      <h2>Sidebar</h2>
      <button onClick={() => setIsModalOpen(true)}>Open Modal</button>
      
      <Modal isOpen={isModalOpen} onClose={() => setIsModalOpen(false)}>
        <h3>Modal Title</h3>
        <p>This modal renders to body, not sidebar!</p>
      </Modal>
    </aside>
  );
}

// Without Portal — Modal would be cropped by Sidebar's overflow
// With Portal — Modal full-screen, no parent constraints
```

Custom container element:

```tsx
function CustomPortalTarget({ children }: { children: React.ReactNode }) {
  // Custom container — yoki existing element, yoki dynamically created
  const target = document.getElementById('portal-root');
  if (!target) return null;
  
  return createPortal(children, target);
}

// HTML setup:
// <body>
//   <div id="root"></div>
//   <div id="portal-root"></div>  ← Modal/Tooltip target
// </body>

function App() {
  return (
    <div id="app">
      <CustomPortalTarget>
        <Modal>...</Modal>
      </CustomPortalTarget>
    </div>
  );
}
```

Multiple containers — alohida portal target'lar:

```tsx
function ModalContainer({ children }: { children: React.ReactNode }) {
  const target = document.getElementById('modal-root');
  return target ? createPortal(children, target) : null;
}

function TooltipContainer({ children }: { children: React.ReactNode }) {
  const target = document.getElementById('tooltip-root');
  return target ? createPortal(children, target) : null;
}

function NotificationContainer({ children }: { children: React.ReactNode }) {
  const target = document.getElementById('notification-root');
  return target ? createPortal(children, target) : null;
}
```

Har portal turi alohida container — z-index management oson.

</details>

---

## `createPortal` API

### Nazariya

`createPortal` — `react-dom` paketidan import qilinadigan function:

```tsx
import { createPortal } from 'react-dom';

createPortal(
  children: React.ReactNode,
  container: Element | DocumentFragment,
  key?: string | null
): React.ReactPortal
```

Argumentlar:

| Parameter | Tip | Tavsif |
|-----------|-----|--------|
| `children` | ReactNode | Portal ichidagi JSX |
| `container` | Element | DOM element (target) — render joyi |
| `key` | string \| null (optional) | React key (siblings orasida) |

Return — React Portal element. JSX render output sifatida ishlatiladi.

```tsx
function ParentWithPortal() {
  return (
    <div className="parent">
      <p>Inside parent</p>
      {createPortal(
        <p>Inside body, but logically here</p>,
        document.body
      )}
    </div>
  );
}
```

`createPortal` `null`/`undefined` qaytarmaydi — har doim React element. Lekin `container` `null` bo'lsa, JSX'da `{container && createPortal(...)}` pattern.

NIMA UCHUN `key` parameter:

```tsx
function NotificationList({ messages }: { messages: string[] }) {
  return (
    <>
      {messages.map((msg, i) => 
        createPortal(
          <div className="notification">{msg}</div>,
          document.body,
          `notif-${i}`  // ✅ key for siblings
        )
      )}
    </>
  );
}
```

Multiple Portals bir parent'da — `key` Reconciliation uchun kerak (cross-ref [`04-reconciliation.md`](04-reconciliation.md)).

QANDAY ISHLAYDI: `createPortal` chaqirilganda — React Element yaratiladi (`$$typeof: REACT_PORTAL_TYPE`). React Reconciler bu element'ni boshqa Fiber tag (`HostPortal`) bilan markup qiladi. Render paytida children container'ga DOM API orqali insert qilinadi.

```tsx
// JSX
createPortal(<div>Hello</div>, document.body, 'my-key');

// React Element output
{
  $$typeof: REACT_PORTAL_TYPE,  // Symbol.for('react.portal')
  key: 'my-key',
  children: <div>Hello</div>,
  containerInfo: document.body,
  implementation: null,
}
```

`containerInfo` — target DOM element. React Reconciler render paytida `containerInfo` ichiga children'ni mount qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

`createPortal` cheklov'lari:

1. **`container` mavjud bo'lishi shart** — render paytida. Aks holda runtime error.
2. **`container` HTML element bo'lishi shart** — `Element` yoki `DocumentFragment`. Text node yoki Comment node taqiq.
3. **Same-origin iframe** — iframe'ning `contentDocument`'iga portal render qilish mumkin agar same-origin bo'lsa va React event system shu document'ga moslangan bo'lsa. Cross-origin iframe — DOM access browser security tomonidan bloklanadi, portal mumkin emas. Default React event delegation parent document'ga ulangan, shu sabab iframe'ga portal noodatiy va edge case.

Container null check pattern:

```tsx
function SafePortal({ children, target }: { children: React.ReactNode; target: Element | null }) {
  if (!target) return null;
  return createPortal(children, target);
}
```

`useEffect` orqali container yaratish:

```tsx
function DynamicPortal({ children }: { children: React.ReactNode }) {
  const [container, setContainer] = useState<HTMLElement | null>(null);
  
  useEffect(() => {
    const div = document.createElement('div');
    div.className = 'portal-container';
    document.body.appendChild(div);
    setContainer(div);
    
    return () => {
      document.body.removeChild(div);
    };
  }, []);
  
  if (!container) return null;
  return createPortal(children, container);
}
```

Dynamic container — har komponent o'z DOM target'i bilan. Cleanup paytida container DOM'dan olib tashlanadi (memory leak oldini olish).

R19 — `createPortal` API o'zgarmagan, lekin `<title>`/`<meta>`/`<link>` tag'lar Portal'siz `<head>`'ga hoist qilinadi (cross-ref [`37-react-19-document-apis.md`](37-react-19-document-apis.md)). Bu Portal use case'larining bir qismini almashtiradi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

API variants:

```tsx
import { createPortal } from 'react-dom';

// 1. Static container — document.body
createPortal(<div>Hello</div>, document.body);

// 2. Element by ID
const target = document.getElementById('modal-root');
target && createPortal(<div>Modal</div>, target);

// 3. With key (multiple portals)
const portals = items.map((item, i) => 
  createPortal(<Item data={item} />, container, `item-${i}`)
);

// 4. Conditional rendering
function Tooltip({ visible, children }: { visible: boolean; children: React.ReactNode }) {
  if (!visible) return null;
  return createPortal(<div className="tooltip">{children}</div>, document.body);
}
```

Custom hook for portal:

```tsx
import React, { useState, useEffect } from 'react';
import { createPortal } from 'react-dom';

function usePortal(targetId: string = 'portal-root'): {
  Portal: (props: { children: React.ReactNode }) => React.ReactElement | null;
  isReady: boolean;
} {
  const [container, setContainer] = useState<HTMLElement | null>(null);
  
  useEffect(() => {
    let target = document.getElementById(targetId);
    let createdNew = false;
    
    if (!target) {
      target = document.createElement('div');
      target.id = targetId;
      document.body.appendChild(target);
      createdNew = true;
    }
    
    setContainer(target);
    
    return () => {
      if (createdNew && target?.parentElement) {
        target.parentElement.removeChild(target);
      }
    };
  }, [targetId]);
  
  const Portal = React.useCallback(
    ({ children }: { children: React.ReactNode }) => {
      if (!container) return null;
      return createPortal(children, container);
    },
    [container]
  );
  
  return { Portal, isReady: container !== null };
}

// Usage
function App() {
  const { Portal, isReady } = usePortal('app-modal-root');
  const [isModalOpen, setIsModalOpen] = useState(false);
  
  return (
    <>
      <button onClick={() => setIsModalOpen(true)}>Open</button>
      {isReady && isModalOpen && (
        <Portal>
          <div className="modal">Modal content</div>
        </Portal>
      )}
    </>
  );
}
```

Custom hook — Portal target dynamic management (yaratish + cleanup).

</details>

---

## DOM Tree vs React Tree — Fundamental Farq

### Nazariya

Portal'larning **eng muhim xususiyati** — DOM tree va React tree o'rtasidagi farq:

- **DOM tree** — browser HTML elements'i (visible structure).
- **React tree** — React komponent hierarchy'si (logical structure).

Portal **ikki tree'ni ajratadi**:

```tsx
function App() {
  return (
    <div className="app">
      <Sidebar>
        <Modal>             {/* React tree: Sidebar > Modal */}
          <p>Modal text</p>
        </Modal>
      </Sidebar>
    </div>
  );
}

// DOM tree:
// <body>
//   <div class="app">
//     <aside class="sidebar">
//       (Modal elementlar bu yerda EMAS)
//     </aside>
//   </div>
//   <div class="modal-backdrop">     ← Portal jumped here
//     <div class="modal">
//       <p>Modal text</p>
//     </div>
//   </div>
// </body>

// React tree:
// App
// └── Sidebar
//     └── Modal              ← React tree'da Sidebar parent
//         └── <p>Modal text</p>
```

NIMA UCHUN bu farq fundamental:

1. **Context inheritance** — React tree bo'ylab. Portal ichidagi component'lar parent Context'larga kira oladi.
2. **Event bubbling** — React tree bo'ylab (DOM emas). Portal'dagi event React parent'larga bubble.
3. **Error boundaries** — React tree bo'ylab. Portal ichidagi xato React parent boundary'ga ko'tariladi.
4. **Reconciliation** — React tree bo'ylab. Parent re-render Portal children'ni ham re-render qiladi.

```tsx
// Context inheritance
const ThemeContext = createContext('light');

function App() {
  return (
    <ThemeContext.Provider value="dark">
      <Sidebar>
        <Modal>
          <ThemedButton />  {/* useContext returns 'dark' */}
        </Modal>
      </Sidebar>
    </ThemeContext.Provider>
  );
}

function ThemedButton() {
  const theme = useContext(ThemeContext);
  return <button className={theme}>Click</button>;
  // ✅ theme === 'dark' — Context React tree bo'ylab uzatiladi
  // Portal DOM jumping ahamiyatsiz — React tree saqlanadi
}
```

QANDAY ISHLAYDI: React Reconciler Portal Fiber'ini boshqa Fiber'lar kabi traverse qiladi. Faqat DOM commit paytida — children ni boshqa container'ga insert qiladi. Logical relationships (parent-child, Context, events) Fiber tree bo'ylab.

```javascript
// React tree (Fiber):
// App.child = Sidebar Fiber
// Sidebar.child = Modal Fiber (Portal type)
// Modal.containerInfo = document.body
// Modal.child = <div className="modal-backdrop"> Fiber

// DOM tree:
// document.body.children = [<div id="app">, ..., <div className="modal-backdrop">]
// <div className="modal-backdrop"> NOT inside <aside class="sidebar">

// React renders, traverses Fiber tree (App → Sidebar → Modal → backdrop)
// DOM commits — appendChild to containerInfo (body), not to Sidebar's DOM
```

<details>
<summary><strong>Under the Hood</strong></summary>

Fiber Portal type — `tag: HostPortal`:

```javascript
// react-reconciler/src/ReactFiber.js (simplified)
const FiberTags = {
  FunctionComponent: 0,
  ClassComponent: 1,
  HostRoot: 3,
  HostPortal: 4,        // ← Portal Fiber tag
  HostComponent: 5,     // div, span, etc.
  HostText: 6,
  // ...
};
```

Portal Fiber `containerInfo` — DOM target element:

```javascript
function createFiberFromPortal(element) {
  return {
    tag: HostPortal,
    type: null,
    pendingProps: element.children,
    stateNode: {
      containerInfo: element.containerInfo,
      pendingChildren: null,
      implementation: element.implementation,
    },
    // ... other Fiber fields
  };
}
```

`stateNode.containerInfo` — DOM element, render paytida children shu yerga insert qilinadi.

Reconciliation Portal:

```javascript
function reconcileChildrenForPortal(workInProgress) {
  const containerInfo = workInProgress.stateNode.containerInfo;
  
  // Push container — child Fibers'lar shu container'ga DOM commit qilinadi
  pushHostContainer(workInProgress, containerInfo);
  
  // Normal Fiber traversal
  reconcileChildren(workInProgress, workInProgress.pendingProps);
}
```

`pushHostContainer` — host context stack'ga container'ni push qilish. Ichki HostComponent'lar (div, span) shu container'ga commit qilinadi.

Memory layout:

```
React Fiber Tree:
  App Fiber
    │
    ├─ Sidebar Fiber
    │    │
    │    └─ Modal Fiber (HostPortal, containerInfo: body)
    │         │
    │         └─ <div backdrop> Fiber
    │              └─ <div modal> Fiber
    │                   └─ <p> Fiber

DOM Tree (after commit):
  <body>
    <div id="app">
      <aside class="sidebar">  (Sidebar's DOM)
        (no modal here)
      </aside>
    </div>
    <div class="modal-backdrop">  (Portal commit'da body'ga ulandi)
      <div class="modal">
        <p>...</p>
      </div>
    </div>
  </body>
```

Fiber tree va DOM tree alohida structures. Portal — bridge ikkalasi orasida.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Context inheritance through Portal:

```tsx
import React, { createContext, useContext } from 'react';
import { createPortal } from 'react-dom';

interface Theme {
  primary: string;
  background: string;
}

const ThemeContext = createContext<Theme>({ primary: '#fff', background: '#000' });

function App() {
  const darkTheme: Theme = { primary: '#fff', background: '#000' };
  
  return (
    <ThemeContext.Provider value={darkTheme}>
      <Sidebar>
        <Modal>
          <ThemedContent />  {/* Inherits dark theme */}
        </Modal>
      </Sidebar>
    </ThemeContext.Provider>
  );
}

function Modal({ children }: { children: React.ReactNode }) {
  return createPortal(
    <div className="modal-backdrop">
      <div className="modal">{children}</div>
    </div>,
    document.body
  );
}

function ThemedContent() {
  const theme = useContext(ThemeContext);
  // ✅ theme === darkTheme — Context normal ishlaydi
  return (
    <div style={{ background: theme.background, color: theme.primary }}>
      Themed in modal
    </div>
  );
}
```

Error Boundary through Portal:

```tsx
function App() {
  return (
    <ErrorBoundary fallback={<p>Error caught</p>}>
      <Sidebar>
        <Modal>
          <BuggyComponent />  {/* Throws error */}
        </Modal>
      </Sidebar>
    </ErrorBoundary>
  );
}

function BuggyComponent() {
  throw new Error('Test error');
}

// ✅ ErrorBoundary catches the error
// React tree'da Modal Sidebar > App ichida — boundary works
// DOM jumping ahamiyatsiz
```

Reconciliation through Portal:

```tsx
function App() {
  const [count, setCount] = useState(0);
  
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
      <Modal>
        <p>Count from parent: {count}</p>  {/* ✅ Re-renders on state change */}
      </Modal>
    </>
  );
}
```

Parent re-render → Portal children re-render (React tree-based reconciliation).

</details>

---

## Event Bubbling React Tree Bo'ylab

### Nazariya

**Eng muhim Portal advanced detail** — event bubbling **React tree** bo'ylab, **DOM tree EMAS**.

```tsx
function App() {
  const handleClick = () => {
    console.log('App clicked');
  };
  
  return (
    <div onClick={handleClick}>
      <Modal>
        <button>Click me</button>  {/* Click event bubbles to App */}
      </Modal>
    </div>
  );
}

// User clicks <button> inside Modal:
// 1. button click (DOM)
// 2. button click (React) — handler attached?
// 3. modal-backdrop click (React) — Modal's handler
// 4. App click (React) — handleClick fires
//
// DOM bubbling:
// 1. button click
// 2. modal-backdrop click
// 3. body click (Portal target)
// 4. document click
//
// React event delegation captures all events at root,
// then dispatches to React tree handlers (parent chain bo'ylab)
```

NIMA UCHUN bu surprising:

- **DOM bubbling** — Portal `<body>`'gacha bubble qiladi. `<App>` outside body'da yo'q.
- **React bubbling** — App komponent'gacha bubble qiladi (React tree parent).

Bu **expected behavior** Portal mexanizmida. Lekin event handler'lar yozayotgan developer chalkashishi mumkin:

```tsx
function App() {
  return (
    <div onClick={() => console.log('App click')}>
      <button>Outside Portal</button>
      <Modal>
        <button>Inside Portal</button>
      </Modal>
    </div>
  );
}
// Outside button click → "App click"
// Inside button click → "App click" (React tree bubbling)
```

QANDAY ISHLAYDI: React event delegation (R17+):

1. R17+ event listener'lar root container'ga (App root)'ga attach qilinadi (cross-ref [`13-event-handling.md`](13-event-handling.md)).
2. DOM event firing → React captures at root.
3. React **target Fiber** topadi (event.target → Fiber).
4. Fiber'dan **React tree parent chain** bo'ylab handler'lar dispatch qilinadi.
5. Portal Fiber `return` pointer'i parent komponent'iga ulangan — bubble parent'larga.

```javascript
// React internal (simplified)
function dispatchEvent(event, targetFiber) {
  let fiber = targetFiber;
  while (fiber !== null) {
    const handler = fiber.memoizedProps?.[`on${event.type}`];
    if (handler && !event.cancelBubble) {
      handler(event);
    }
    fiber = fiber.return;  // ← Parent Fiber (React tree)
  }
}
```

`fiber.return` — Portal Fiber'da parent komponent'i (Sidebar / App), DOM parent'i emas (body).

`stopPropagation()` — bubbling to'xtatish:

```tsx
function Modal({ children }: { children: React.ReactNode }) {
  return createPortal(
    <div onClick={(e) => e.stopPropagation()}>  {/* ← block bubbling */}
      {children}
    </div>,
    document.body
  );
}

function App() {
  return (
    <div onClick={() => console.log('App')}>
      <Modal>
        <button>Click</button>
      </Modal>
    </div>
  );
}
// Click button → Modal's stopPropagation → App click NOT fired
```

NIMA UCHUN: Modal'da `onClick={onClose}` qo'yish backdrop click yopish uchun, lekin Modal **content** click yopmaslik kerak. `stopPropagation` content click'ini block qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

R17+ event delegation root container'ga (cross-ref [`13-event-handling.md`](13-event-handling.md)):

```javascript
// react-dom-bindings/src/events/DOMPluginEventSystem.js (simplified)
function listenToAllSupportedEvents(rootContainerElement) {
  allNativeEvents.forEach(domEventName => {
    rootContainerElement.addEventListener(domEventName, (nativeEvent) => {
      dispatchEventForPluginEventSystem(nativeEvent, rootContainerElement);
    });
  });
}

function dispatchEventForPluginEventSystem(nativeEvent, rootContainerElement) {
  const targetInst = getInstanceFromNode(nativeEvent.target);
  // targetInst — Fiber for the target DOM element
  
  // Traverse Fiber tree (React tree, not DOM tree)
  let listeners = collectListenersForFiber(targetInst, nativeEvent.type);
  
  // Dispatch in parent chain order
  for (const listener of listeners) {
    listener.handler(syntheticEvent);
    if (syntheticEvent.isPropagationStopped()) break;
  }
}

function collectListenersForFiber(fiber, eventType) {
  const listeners = [];
  let node = fiber;
  while (node !== null) {
    const handler = node.memoizedProps?.[`on${eventType}`];
    if (handler) {
      listeners.push({ fiber: node, handler });
    }
    node = node.return;  // ← React tree parent (Fiber.return)
  }
  return listeners;
}
```

`fiber.return` — Fiber tree parent (DOM tree emas). Portal Fiber'i komponent parent'iga `return` pointer'i bilan bog'langan — Sidebar/App/whatever.

R16 va eski — event listener `document`'ga attach qilinardi:

```javascript
// R16 — document-level delegation
document.addEventListener('click', (e) => {
  // Find React Fiber for e.target
  // Dispatch handlers
});

// R17+ — root container-level delegation
const root = document.getElementById('root');
root.addEventListener('click', /* ... */);
// Multiple React apps on same page — alohida listener'lar
```

R17+ fix — **microfrontends compatibility** (cross-ref [`13-event-handling.md`](13-event-handling.md)).

R18+ Concurrent rendering — Portal ham concurrent-aware. Suspense Boundary Portal ichidan oshib o'tadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Event bubbling demo:

```tsx
import React from 'react';
import { createPortal } from 'react-dom';

function App() {
  return (
    <div 
      onClick={() => console.log('App clicked')}
      style={{ padding: 20, background: 'lightblue' }}
    >
      <p>App content</p>
      <button>Outside button</button>
      <Modal>
        <button>Inside Portal button</button>
      </Modal>
    </div>
  );
}

function Modal({ children }: { children: React.ReactNode }) {
  return createPortal(
    <div 
      onClick={() => console.log('Modal clicked')}
      style={{ position: 'fixed', top: 0, left: 0, padding: 20, background: 'lightyellow' }}
    >
      <p>Modal content</p>
      {children}
    </div>,
    document.body
  );
}

// Click "Outside button":
// → "App clicked"

// Click "Inside Portal button":
// → "Modal clicked"
// → "App clicked"  (React tree bubbling, NOT DOM)

// DOM tree:
// body
//   ├── #root (App's container)
//   │   └── div (App)
//   │       └── button (Outside)
//   └── div (Modal — Portal'd)
//       └── button (Inside)

// React tree:
// App
//   ├── button (Outside)
//   └── Modal
//       └── div (Modal content)
//           └── button (Inside)
```

`stopPropagation` to block bubbling:

```tsx
function Modal({ isOpen, onClose, children }: { 
  isOpen: boolean; 
  onClose: () => void; 
  children: React.ReactNode;
}) {
  if (!isOpen) return null;
  
  return createPortal(
    <div 
      className="modal-backdrop"
      onClick={onClose}  // Close on backdrop click
    >
      <div 
        className="modal"
        onClick={(e) => e.stopPropagation()}  // ← Don't bubble to backdrop
      >
        {children}
        <button onClick={onClose}>Close</button>
      </div>
    </div>,
    document.body
  );
}

function App() {
  const [isOpen, setIsOpen] = useState(false);
  
  return (
    <div onClick={() => console.log('App click')}>  {/* Won't fire from modal content */}
      <button onClick={() => setIsOpen(true)}>Open Modal</button>
      <Modal isOpen={isOpen} onClose={() => setIsOpen(false)}>
        <h2>Modal Title</h2>
        <p>Click outside to close, or click inside — won't close</p>
      </Modal>
    </div>
  );
}
```

Custom event capture vs bubble:

```tsx
function App() {
  return (
    <div 
      onClickCapture={(e) => console.log('App capture')}  // Capture phase
      onClick={() => console.log('App bubble')}              // Bubble phase
    >
      <Modal>
        <button onClick={() => console.log('Button click')}>Click</button>
      </Modal>
    </div>
  );
}

// Click sequence:
// 1. "App capture"     (capture phase, top-down)
// 2. "Button click"    (target)
// 3. "App bubble"      (bubble phase, bottom-up — React tree)
```

R16+ React capture phase ham qo'llab-quvvatlaydi (`onClickCapture`, `onMouseOverCapture`, va h.k.).

</details>

---

## Real-World: Modal Implementation

### Nazariya

> **Modern alternative:** R19 + modern browser context'da **HTML native `<dialog>` element + `.showModal()`** ko'p holatda afzal — top layer (z-index'siz), browser-native ESC handling, focus trap (Chrome/Firefox/Safari'ning so'nggi versiyalarida), `::backdrop` CSS pseudo-element. Portal-based Modal — animation flexibility yoki `<dialog>` mavjud bo'lmagan browser fallback uchun. Tafsilot pastda "[z-index va Stacking Contexts](#z-index-va-stacking-contexts)" bo'limida.

Modal — Portal'ning eng klassik use case. Production-grade Portal-based Modal:

1. **Portal** — `<body>`'ga render (overflow/z-index/transform escape).
2. **Backdrop click close** — outside click yopish.
3. **Escape key close** — keyboard accessibility (manual listener — `<dialog>`'da native).
4. **Focus trap** — Tab key Modal ichida cycle (manual — `<dialog>` modern browser'da native).
5. **Focus return** — close paytida triggering element'ga focus qaytarish.
6. **Body scroll lock** — Modal ochiq paytida arqa scroll bloklash.
7. **ARIA** — `role="dialog"`, `aria-modal="true"`, `aria-labelledby`.
8. **Animation** — open/close transitions.

API design:

```tsx
<Modal 
  isOpen={isOpen} 
  onClose={() => setIsOpen(false)}
  title="Confirmation"
>
  <p>Are you sure?</p>
  <button onClick={handleConfirm}>Confirm</button>
</Modal>
```

QANDAY ISHLAYDI: Modal komponent:

1. `isOpen === false` → `null` qaytaradi.
2. `isOpen === true` → `createPortal` body'ga render.
3. `useEffect` — body scroll lock + Escape listener + focus trap setup.
4. Cleanup paytida — restore scroll + remove listeners + restore focus.

NIMA UCHUN production-grade Modal'da bu hammasi:

| Feature | Sabab |
|---------|-------|
| Portal | Parent constraints'dan qutulish |
| Backdrop close | UX expectation |
| Escape close | Keyboard a11y |
| Focus trap | Keyboard navigation Modal ichida |
| Focus return | A11y best practice (focus context preservation) |
| Body scroll lock | Modal kontent ochiq, background scroll noto'g'ri UX |
| ARIA | Screen reader support |
| Animation | UX polish |

<details>
<summary><strong>Under the Hood</strong></summary>

Modal lifecycle:

```
1. User clicks "Open Modal" button
   → setIsOpen(true)

2. Modal component renders:
   - createPortal(modalUI, body)
   - Mounted to DOM

3. useEffect setup:
   - body.style.overflow = 'hidden' (scroll lock)
   - document.addEventListener('keydown', escapeHandler)
   - focus first focusable inside modal
   - save previous focus (document.activeElement)

4. User interacts inside modal (Tab key, etc.)
   - Focus trap keeps focus inside

5. User presses Escape OR clicks backdrop
   → onClose() → setIsOpen(false)

6. Modal unmounts:
   - useEffect cleanup runs
   - body.style.overflow = '' (restore scroll)
   - removeEventListener('keydown')
   - restore previous focus (focus return)
```

Body scroll lock cross-platform issue:

```tsx
// ⚠️ Eski iOS Safari (pre-15) momentum scroll'ni hammasini bloklamaydi
document.body.style.overflow = 'hidden';

// ✅ Universal — position: fixed scroll position'ni ham saqlaydi
function lockBodyScroll() {
  const scrollY = window.scrollY;
  document.body.style.position = 'fixed';
  document.body.style.top = `-${scrollY}px`;
  document.body.style.width = '100%';
}

function unlockBodyScroll() {
  const scrollY = document.body.style.top;
  document.body.style.position = '';
  document.body.style.top = '';
  document.body.style.width = '';
  window.scrollTo(0, parseInt(scrollY || '0') * -1);
}
```

iOS Safari'ning eski versiyalari (pre-15.4) `overflow: hidden` momentum scroll'ni to'liq bloklamaydi. Joriy iOS (16+) ko'p holatda to'g'ri ishlaydi, lekin `position: fixed` workaround universal — har platforma va versiyada scroll position'ni saqlab tiklaydi. Production'da `body-scroll-lock` library tavsiya etiladi (edge case'larni qamrab oladi).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Production-grade Modal:

```tsx
import React, { useEffect, useRef, useId } from 'react';
import { createPortal } from 'react-dom';

interface ModalProps {
  isOpen: boolean;
  onClose: () => void;
  title?: string;
  children: React.ReactNode;
}

export function Modal({ isOpen, onClose, title, children }: ModalProps) {
  const modalRef = useRef<HTMLDivElement>(null);
  const previousFocusRef = useRef<HTMLElement | null>(null);
  const titleId = useId();
  
  useEffect(() => {
    if (!isOpen) return;
    
    // Save current focus
    previousFocusRef.current = document.activeElement as HTMLElement;
    
    // Body scroll lock
    const originalOverflow = document.body.style.overflow;
    document.body.style.overflow = 'hidden';
    
    // Focus first focusable in modal
    const focusableSelectors = 
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])';
    const focusables = modalRef.current?.querySelectorAll<HTMLElement>(focusableSelectors);
    focusables?.[0]?.focus();
    
    // Escape key handler
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === 'Escape') {
        onClose();
      }
    };
    
    // Focus trap
    const handleFocusTrap = (e: KeyboardEvent) => {
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
    
    document.addEventListener('keydown', handleKeyDown);
    document.addEventListener('keydown', handleFocusTrap);
    
    return () => {
      // Cleanup
      document.body.style.overflow = originalOverflow;
      document.removeEventListener('keydown', handleKeyDown);
      document.removeEventListener('keydown', handleFocusTrap);
      
      // Restore focus
      previousFocusRef.current?.focus();
    };
  }, [isOpen, onClose]);
  
  if (!isOpen) return null;
  
  return createPortal(
    <div 
      className="modal-backdrop"
      onClick={onClose}
      role="presentation"
    >
      <div 
        ref={modalRef}
        className="modal"
        role="dialog"
        aria-modal="true"
        aria-labelledby={title ? titleId : undefined}
        onClick={(e) => e.stopPropagation()}
      >
        {title && (
          <header className="modal-header">
            <h2 id={titleId}>{title}</h2>
            <button 
              type="button"
              onClick={onClose} 
              aria-label="Close modal"
              className="modal-close"
            >
              ×
            </button>
          </header>
        )}
        <div className="modal-body">
          {children}
        </div>
      </div>
    </div>,
    document.body
  );
}

// Usage
function ConfirmDelete() {
  const [isOpen, setIsOpen] = React.useState(false);
  
  return (
    <>
      <button onClick={() => setIsOpen(true)}>Delete Item</button>
      
      <Modal 
        isOpen={isOpen} 
        onClose={() => setIsOpen(false)}
        title="Confirm Deletion"
      >
        <p>Are you sure you want to delete this item?</p>
        <p>This action cannot be undone.</p>
        
        <div className="modal-actions">
          <button onClick={() => setIsOpen(false)}>Cancel</button>
          <button 
            onClick={() => {
              // Delete logic
              setIsOpen(false);
            }}
            className="danger"
          >
            Delete
          </button>
        </div>
      </Modal>
    </>
  );
}
```

iOS-safe scroll lock:

```tsx
function useBodyScrollLock(locked: boolean) {
  useEffect(() => {
    if (!locked) return;
    
    const isIOS = /iPad|iPhone|iPod/.test(navigator.userAgent);
    
    if (isIOS) {
      const scrollY = window.scrollY;
      document.body.style.position = 'fixed';
      document.body.style.top = `-${scrollY}px`;
      document.body.style.left = '0';
      document.body.style.right = '0';
      
      return () => {
        const top = document.body.style.top;
        document.body.style.position = '';
        document.body.style.top = '';
        document.body.style.left = '';
        document.body.style.right = '';
        window.scrollTo(0, parseInt(top || '0') * -1);
      };
    } else {
      const original = document.body.style.overflow;
      document.body.style.overflow = 'hidden';
      return () => {
        document.body.style.overflow = original;
      };
    }
  }, [locked]);
}
```

Compound Components Modal (cross-ref [`26-compound-components.md`](26-compound-components.md)):

```tsx
const ModalContext = createContext<{ onClose: () => void } | null>(null);

function ModalRoot({ isOpen, onClose, children }: { 
  isOpen: boolean; 
  onClose: () => void; 
  children: React.ReactNode;
}) {
  if (!isOpen) return null;
  
  return createPortal(
    <ModalContext.Provider value={{ onClose }}>
      <div className="modal-backdrop" onClick={onClose}>
        <div 
          className="modal" 
          role="dialog" 
          aria-modal="true"
          onClick={(e) => e.stopPropagation()}
        >
          {children}
        </div>
      </div>
    </ModalContext.Provider>,
    document.body
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

function ModalCloseButton(props: React.ButtonHTMLAttributes<HTMLButtonElement>) {
  const ctx = useContext(ModalContext);
  return <button {...props} onClick={ctx?.onClose} aria-label="Close" />;
}

const Modal = Object.assign(ModalRoot, {
  Header: ModalHeader,
  Body: ModalBody,
  Footer: ModalFooter,
  Close: ModalCloseButton,
});

// Usage
<Modal isOpen={isOpen} onClose={onClose}>
  <Modal.Header>
    <h2>Title</h2>
    <Modal.Close>×</Modal.Close>
  </Modal.Header>
  <Modal.Body>
    <p>Content</p>
  </Modal.Body>
  <Modal.Footer>
    <Modal.Close>Cancel</Modal.Close>
    <button onClick={confirm}>OK</button>
  </Modal.Footer>
</Modal>
```

</details>

---

## Real-World: Tooltip Implementation

### Nazariya

Tooltip — element atrofida hover yoki focus paytida ko'rinadigan kichik info popup. Portal `<body>`'ga render — `overflow: hidden` parent'lardan qutulish va viewport-relative positioning.

API:

```tsx
<Tooltip content="More info about this">
  <button>Hover me</button>
</Tooltip>
```

Asosiy elementlar:

1. **Trigger** — children element (hover/focus listener).
2. **Tooltip content** — Portal'da `<body>` ostida.
3. **Position** — trigger element position'iga qarab calculate.
4. **Arrow** — visual indicator (optional).

State:
- `isVisible: boolean` — tooltip ko'rinadimi.
- `position: { top, left }` — tooltip joylashuvi.

Behavior:

- **Mouse enter trigger** → show tooltip + calculate position.
- **Mouse leave trigger** → hide tooltip (with delay for UX).
- **Focus trigger** → show tooltip (keyboard accessibility).
- **Blur trigger** → hide tooltip.
- **Window resize** → recalculate position.
- **Window scroll** → recalculate position (yoki hide).

NIMA UCHUN Portal:

- Trigger element `overflow: hidden` parent ichida bo'lishi mumkin.
- Tooltip viewport edge'da (right/bottom) bo'lsa — hidden ichida cropped.
- Portal `<body>` — viewport-relative positioning.

Position calculation:

```tsx
function calculateTooltipPosition(
  trigger: HTMLElement, 
  placement: 'top' | 'bottom' | 'left' | 'right'
): { top: number; left: number } {
  const rect = trigger.getBoundingClientRect();
  
  switch (placement) {
    case 'top':
      return {
        top: rect.top - 8,  // 8px gap
        left: rect.left + rect.width / 2,
      };
    case 'bottom':
      return {
        top: rect.bottom + 8,
        left: rect.left + rect.width / 2,
      };
    // ... other placements
  }
}
```

`getBoundingClientRect()` — sync DOM read, viewport-relative coordinates. `useLayoutEffect` ichida (cross-ref [`17-uselayouteffect.md`](17-uselayouteffect.md)).

Modern alternative — **`@floating-ui/react`** library — viewport-aware positioning, auto-flip (top → bottom if no space), arrow positioning.

<details>
<summary><strong>Under the Hood</strong></summary>

Tooltip lifecycle:

```
1. Mouse enter trigger
   → setVisible(true)
   → useLayoutEffect: calculate position from trigger.getBoundingClientRect()
   → setPosition({ top, left })

2. Render Portal at calculated position

3. Mouse leave trigger
   → setVisible(false) with setTimeout (delay)
   → Portal unmounts

4. Window resize / scroll
   → ResizeObserver / scroll listener
   → recalculate position
```

Position adjustment for viewport edges:

```tsx
function adjustPosition(
  position: { top: number; left: number; width: number; height: number },
  tooltipDimensions: { width: number; height: number }
): { top: number; left: number } {
  const padding = 8;
  let { top, left } = position;
  
  // Right edge
  if (left + tooltipDimensions.width > window.innerWidth) {
    left = window.innerWidth - tooltipDimensions.width - padding;
  }
  
  // Left edge
  if (left < padding) {
    left = padding;
  }
  
  // Top edge — flip to bottom
  if (top < padding) {
    top = position.top + position.height + padding;
  }
  
  // Bottom edge — flip to top
  if (top + tooltipDimensions.height > window.innerHeight - padding) {
    top = position.top - tooltipDimensions.height - padding;
  }
  
  return { top, left };
}
```

Auto-flip pattern — tooltip viewport edge'da bo'lsa qarama-qarshi tomonga ko'chirish.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Sodda Tooltip:

```tsx
import React, { useState, useRef, useLayoutEffect } from 'react';
import { createPortal } from 'react-dom';

interface TooltipProps {
  content: React.ReactNode;
  placement?: 'top' | 'bottom' | 'left' | 'right';
  children: React.ReactElement;
}

export function Tooltip({ content, placement = 'top', children }: TooltipProps) {
  const [isVisible, setIsVisible] = useState(false);
  const [position, setPosition] = useState({ top: 0, left: 0 });
  const triggerRef = useRef<HTMLElement | null>(null);
  
  const handleMouseEnter = (e: React.MouseEvent) => {
    triggerRef.current = e.currentTarget as HTMLElement;
    setIsVisible(true);
  };
  
  const handleMouseLeave = () => {
    setIsVisible(false);
  };
  
  useLayoutEffect(() => {
    if (!isVisible || !triggerRef.current) return;
    
    const rect = triggerRef.current.getBoundingClientRect();
    
    let top = 0, left = 0;
    switch (placement) {
      case 'top':
        top = rect.top - 8;
        left = rect.left + rect.width / 2;
        break;
      case 'bottom':
        top = rect.bottom + 8;
        left = rect.left + rect.width / 2;
        break;
      case 'left':
        top = rect.top + rect.height / 2;
        left = rect.left - 8;
        break;
      case 'right':
        top = rect.top + rect.height / 2;
        left = rect.right + 8;
        break;
    }
    
    setPosition({ top, left });
  }, [isVisible, placement]);
  
  // Clone trigger element with event handlers
  const trigger = React.cloneElement(children, {
    onMouseEnter: handleMouseEnter,
    onMouseLeave: handleMouseLeave,
    onFocus: handleMouseEnter,
    onBlur: handleMouseLeave,
  });
  
  return (
    <>
      {trigger}
      {isVisible && createPortal(
        <div 
          className={`tooltip tooltip-${placement}`}
          role="tooltip"
          style={{
            position: 'fixed',
            top: position.top,
            left: position.left,
            transform: placement === 'top' || placement === 'bottom' 
              ? 'translateX(-50%)' 
              : 'translateY(-50%)',
          }}
        >
          {content}
        </div>,
        document.body
      )}
    </>
  );
}

// Usage
<Tooltip content="Click to delete this item" placement="top">
  <button>Delete</button>
</Tooltip>
```

Production with delay + window resize:

```tsx
import React, { useState, useRef, useLayoutEffect, useEffect } from 'react';

export function ProductionTooltip({ content, placement = 'top', children, delay = 200 }: TooltipProps & { delay?: number }) {
  const [isVisible, setIsVisible] = useState(false);
  const [position, setPosition] = useState({ top: 0, left: 0 });
  const triggerRef = useRef<HTMLElement | null>(null);
  const showTimerRef = useRef<ReturnType<typeof setTimeout> | null>(null);
  const hideTimerRef = useRef<ReturnType<typeof setTimeout> | null>(null);
  
  const showTooltip = (target: HTMLElement) => {
    triggerRef.current = target;
    if (hideTimerRef.current) clearTimeout(hideTimerRef.current);
    showTimerRef.current = setTimeout(() => setIsVisible(true), delay);
  };
  
  const hideTooltip = () => {
    if (showTimerRef.current) clearTimeout(showTimerRef.current);
    hideTimerRef.current = setTimeout(() => setIsVisible(false), 100);
  };
  
  const calculatePosition = () => {
    if (!triggerRef.current) return;
    const rect = triggerRef.current.getBoundingClientRect();
    
    // ... same as basic version
    setPosition(/* calculated */);
  };
  
  useLayoutEffect(() => {
    if (isVisible) calculatePosition();
  }, [isVisible, placement]);
  
  // Recalculate on window resize / scroll
  useEffect(() => {
    if (!isVisible) return;
    
    const handler = () => calculatePosition();
    window.addEventListener('resize', handler);
    window.addEventListener('scroll', handler, true);  // capture for nested scroll
    
    return () => {
      window.removeEventListener('resize', handler);
      window.removeEventListener('scroll', handler, true);
    };
  }, [isVisible]);
  
  // Cleanup timers
  useEffect(() => {
    return () => {
      if (showTimerRef.current) clearTimeout(showTimerRef.current);
      if (hideTimerRef.current) clearTimeout(hideTimerRef.current);
    };
  }, []);
  
  const trigger = React.cloneElement(children, {
    onMouseEnter: (e: React.MouseEvent) => showTooltip(e.currentTarget as HTMLElement),
    onMouseLeave: hideTooltip,
    onFocus: (e: React.FocusEvent) => showTooltip(e.currentTarget as HTMLElement),
    onBlur: hideTooltip,
  });
  
  return (
    <>
      {trigger}
      {isVisible && createPortal(
        <div className="tooltip" role="tooltip" style={{ position: 'fixed', ...position }}>
          {content}
        </div>,
        document.body
      )}
    </>
  );
}
```

Production'da `@floating-ui/react` library afzal — auto-flip, viewport awareness, arrow positioning, accessibility (aria-describedby).

</details>

---

## Real-World: Dropdown va Popover

### Nazariya

Dropdown menu va Popover — Tooltip'ga o'xshash, lekin **interactive content**. Click trigger → open, click outside → close.

API:

```tsx
<Dropdown 
  trigger={<button>Menu ▼</button>}
  content={
    <ul>
      <li onClick={() => navigate('/profile')}>Profile</li>
      <li onClick={() => navigate('/settings')}>Settings</li>
      <li onClick={() => logout()}>Logout</li>
    </ul>
  }
/>
```

Behavior:

- **Click trigger** → toggle open.
- **Click outside** → close (via `useOnClickOutside` cross-ref [`24-custom-hooks.md`](24-custom-hooks.md)).
- **Escape key** → close.
- **Position** — trigger position'i ostida (yoki adjustable).
- **ARIA** — `aria-haspopup`, `aria-expanded`, `aria-controls`.

NIMA UCHUN Portal: parent overflow constraints, z-index stacking, positioning fixed (viewport-relative).

QANDAY ISHLAYDI:

1. Click trigger → `setIsOpen(true)`.
2. Calculate position from trigger `getBoundingClientRect`.
3. Render Portal'da popup content.
4. `useOnClickOutside` listener → outside click yopish.
5. Escape key listener → yopish.
6. Render trigger normally (Portal'siz).

<details>
<summary><strong>Under the Hood</strong></summary>

Dropdown vs Tooltip farq:

| Aspect | Tooltip | Dropdown |
|--------|---------|----------|
| Trigger | Hover/Focus | Click |
| Content | Plain text | Interactive (buttons, links) |
| Persistence | Hover-bound | Persistent until close |
| Close | Mouse leave | Click outside / Escape |
| ARIA | `role="tooltip"` | `aria-haspopup="true"` |

Click outside detection (cross-ref [`24-custom-hooks.md`](24-custom-hooks.md)):

```tsx
function useOnClickOutside(
  ref: React.RefObject<HTMLElement>,
  handler: (e: MouseEvent) => void
) {
  useEffect(() => {
    const listener = (e: MouseEvent) => {
      if (!ref.current || ref.current.contains(e.target as Node)) return;
      handler(e);
    };
    document.addEventListener('mousedown', listener);
    return () => document.removeEventListener('mousedown', listener);
  }, [ref, handler]);
}
```

Portal Edge case — `ref.current.contains` portal ichidagi click'larni "outside" deb hisoblaydi (Portal DOM ko'chgan). Yechim:

```tsx
function useOnClickOutsidePortal(
  triggerRef: React.RefObject<HTMLElement>,
  popupRef: React.RefObject<HTMLElement>,
  handler: (e: MouseEvent) => void
) {
  useEffect(() => {
    const listener = (e: MouseEvent) => {
      const target = e.target as Node;
      
      // Inside trigger?
      if (triggerRef.current?.contains(target)) return;
      
      // Inside popup (Portal'd)?
      if (popupRef.current?.contains(target)) return;
      
      handler(e);
    };
    document.addEventListener('mousedown', listener);
    return () => document.removeEventListener('mousedown', listener);
  }, [triggerRef, popupRef, handler]);
}
```

Trigger ref + popup ref — ikkalasi ham contains check.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Dropdown implementation:

```tsx
import React, { useState, useRef, useLayoutEffect, useEffect } from 'react';
import { createPortal } from 'react-dom';

interface DropdownProps {
  trigger: React.ReactNode;
  children: React.ReactNode;
  placement?: 'bottom-start' | 'bottom-end' | 'top-start' | 'top-end';
}

export function Dropdown({ trigger, children, placement = 'bottom-start' }: DropdownProps) {
  const [isOpen, setIsOpen] = useState(false);
  const [position, setPosition] = useState({ top: 0, left: 0 });
  const triggerRef = useRef<HTMLDivElement>(null);
  const popupRef = useRef<HTMLDivElement>(null);
  
  // Position calculation
  useLayoutEffect(() => {
    if (!isOpen || !triggerRef.current || !popupRef.current) return;
    
    const triggerRect = triggerRef.current.getBoundingClientRect();
    const popupRect = popupRef.current.getBoundingClientRect();
    
    let top = 0, left = 0;
    
    switch (placement) {
      case 'bottom-start':
        top = triggerRect.bottom + 4;
        left = triggerRect.left;
        break;
      case 'bottom-end':
        top = triggerRect.bottom + 4;
        left = triggerRect.right - popupRect.width;
        break;
      case 'top-start':
        top = triggerRect.top - popupRect.height - 4;
        left = triggerRect.left;
        break;
      case 'top-end':
        top = triggerRect.top - popupRect.height - 4;
        left = triggerRect.right - popupRect.width;
        break;
    }
    
    // Adjust for viewport
    if (left + popupRect.width > window.innerWidth) {
      left = window.innerWidth - popupRect.width - 8;
    }
    if (left < 8) left = 8;
    
    setPosition({ top, left });
  }, [isOpen, placement]);
  
  // Click outside
  useEffect(() => {
    if (!isOpen) return;
    
    const listener = (e: MouseEvent) => {
      const target = e.target as Node;
      if (triggerRef.current?.contains(target)) return;
      if (popupRef.current?.contains(target)) return;
      setIsOpen(false);
    };
    
    document.addEventListener('mousedown', listener);
    return () => document.removeEventListener('mousedown', listener);
  }, [isOpen]);
  
  // Escape key
  useEffect(() => {
    if (!isOpen) return;
    
    const handler = (e: KeyboardEvent) => {
      if (e.key === 'Escape') {
        setIsOpen(false);
      }
    };
    
    document.addEventListener('keydown', handler);
    return () => document.removeEventListener('keydown', handler);
  }, [isOpen]);
  
  return (
    <>
      <div ref={triggerRef} onClick={() => setIsOpen(o => !o)}>
        {trigger}
      </div>
      
      {isOpen && createPortal(
        <div
          ref={popupRef}
          className="dropdown-popup"
          role="menu"
          style={{
            position: 'fixed',
            top: position.top,
            left: position.left,
            zIndex: 1000,
          }}
        >
          {children}
        </div>,
        document.body
      )}
    </>
  );
}

// Usage
function UserMenu() {
  return (
    <Dropdown
      placement="bottom-end"
      trigger={
        <button aria-haspopup="true">
          User Menu ▼
        </button>
      }
    >
      <ul className="menu">
        <li><a href="/profile">Profile</a></li>
        <li><a href="/settings">Settings</a></li>
        <li><button onClick={logout}>Logout</button></li>
      </ul>
    </Dropdown>
  );
}
```

Popover with arrow (basic):

```tsx
function Popover({ trigger, children, placement = 'bottom' }: PopoverProps) {
  const [isOpen, setIsOpen] = useState(false);
  const [position, setPosition] = useState({ top: 0, left: 0, arrowLeft: 0 });
  const triggerRef = useRef<HTMLDivElement>(null);
  const popoverRef = useRef<HTMLDivElement>(null);
  
  useLayoutEffect(() => {
    if (!isOpen || !triggerRef.current) return;
    
    const triggerRect = triggerRef.current.getBoundingClientRect();
    const popoverRect = popoverRef.current?.getBoundingClientRect();
    
    if (!popoverRect) return;
    
    const top = triggerRect.bottom + 8;
    let left = triggerRect.left + triggerRect.width / 2 - popoverRect.width / 2;
    
    // Viewport edge adjustment
    if (left < 8) left = 8;
    if (left + popoverRect.width > window.innerWidth - 8) {
      left = window.innerWidth - popoverRect.width - 8;
    }
    
    // Arrow position relative to trigger center
    const triggerCenter = triggerRect.left + triggerRect.width / 2;
    const arrowLeft = triggerCenter - left;
    
    setPosition({ top, left, arrowLeft });
  }, [isOpen]);
  
  return (
    <>
      <div ref={triggerRef} onClick={() => setIsOpen(o => !o)}>
        {trigger}
      </div>
      
      {isOpen && createPortal(
        <div 
          ref={popoverRef}
          className="popover"
          role="dialog"
          style={{ position: 'fixed', top: position.top, left: position.left }}
        >
          <div 
            className="popover-arrow"
            style={{ left: position.arrowLeft }}
          />
          <div className="popover-content">
            {children}
          </div>
        </div>,
        document.body
      )}
    </>
  );
}
```

</details>

---

## Real-World: Toast / Notification System

### Nazariya

Toast / Notification — qisqa muddatli xabarlar (success, error, info). Top-right corner'da yoki bottom-center'da ko'rinadi, avtomatik yo'qoladi.

API:

```tsx
const { toast } = useToast();

toast.success('Saved successfully');
toast.error('Failed to save');
toast.info('Loading...');
```

Implementation strategy:

1. **Context Provider** — global toast state.
2. **Portal container** — `<body>`'ga render (top-right, fixed positioning).
3. **Toast queue** — multiple toasts stack.
4. **Auto-dismiss** — `setTimeout` (3-5 seconds default).
5. **Animation** — entry/exit (slide, fade).
6. **Manual dismiss** — close button.

QANDAY ISHLAYDI:

1. App'ni `ToastProvider` orqali wrap qilinadi.
2. `useToast` hook — `toast.success/error/info` functions.
3. Function chaqirilganda — Context state'ga toast add (`{ id, type, message }`).
4. ToastContainer Portal'da render — har toast'ni list'da ko'rsatadi.
5. Auto-dismiss timer — toast remove from queue.

NIMA UCHUN Portal: top-level positioning (har sahifa atrofida ko'rinadi), z-index management, animation isolation.

<details>
<summary><strong>Under the Hood</strong></summary>

Toast queue management:

```tsx
interface Toast {
  id: string;
  type: 'success' | 'error' | 'info';
  message: string;
  duration?: number;
}

// Reducer pattern
type Action = 
  | { type: 'ADD'; toast: Toast }
  | { type: 'REMOVE'; id: string }
  | { type: 'CLEAR' };

function toastReducer(state: Toast[], action: Action): Toast[] {
  switch (action.type) {
    case 'ADD':
      return [...state, action.toast];
    case 'REMOVE':
      return state.filter(t => t.id !== action.id);
    case 'CLEAR':
      return [];
  }
}
```

Animation strategies:

1. **CSS transitions** — `opacity` + `transform: translateX()`.
2. **`framer-motion`** — `<AnimatePresence>` + `motion.div`.
3. **`react-spring`** — physics-based.

CSS-only example:

```css
.toast {
  opacity: 0;
  transform: translateX(100%);
  transition: opacity 0.3s, transform 0.3s;
}

.toast.entering,
.toast.entered {
  opacity: 1;
  transform: translateX(0);
}

.toast.exiting {
  opacity: 0;
  transform: translateX(100%);
}
```

State machine — `entering` → `entered` → `exiting`. Mount unmount paytida CSS class transition.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

To'liq Toast system:

```tsx
import React, { createContext, useContext, useState, useCallback, useEffect } from 'react';
import { createPortal } from 'react-dom';

type ToastType = 'success' | 'error' | 'info' | 'warning';

interface Toast {
  id: string;
  type: ToastType;
  message: string;
  duration: number;
}

interface ToastContextValue {
  toast: {
    success: (message: string, duration?: number) => void;
    error: (message: string, duration?: number) => void;
    info: (message: string, duration?: number) => void;
    warning: (message: string, duration?: number) => void;
  };
  dismiss: (id: string) => void;
}

const ToastContext = createContext<ToastContextValue | null>(null);

export function useToast() {
  const ctx = useContext(ToastContext);
  if (!ctx) throw new Error('useToast must be used inside <ToastProvider>');
  return ctx;
}

export function ToastProvider({ children }: { children: React.ReactNode }) {
  const [toasts, setToasts] = useState<Toast[]>([]);
  
  const addToast = useCallback((type: ToastType, message: string, duration: number = 4000) => {
    const id = `toast-${Date.now()}-${Math.random().toString(36).slice(2)}`;
    const newToast: Toast = { id, type, message, duration };
    
    setToasts(prev => [...prev, newToast]);
    
    if (duration > 0) {
      setTimeout(() => {
        setToasts(prev => prev.filter(t => t.id !== id));
      }, duration);
    }
  }, []);
  
  const dismiss = useCallback((id: string) => {
    setToasts(prev => prev.filter(t => t.id !== id));
  }, []);
  
  const toast = {
    success: (msg: string, dur?: number) => addToast('success', msg, dur),
    error: (msg: string, dur?: number) => addToast('error', msg, dur),
    info: (msg: string, dur?: number) => addToast('info', msg, dur),
    warning: (msg: string, dur?: number) => addToast('warning', msg, dur),
  };
  
  return (
    <ToastContext.Provider value={{ toast, dismiss }}>
      {children}
      <ToastContainer toasts={toasts} onDismiss={dismiss} />
    </ToastContext.Provider>
  );
}

function ToastContainer({ toasts, onDismiss }: { toasts: Toast[]; onDismiss: (id: string) => void }) {
  if (toasts.length === 0) return null;
  
  return createPortal(
    <div 
      className="toast-container"
      style={{
        position: 'fixed',
        top: 16,
        right: 16,
        zIndex: 9999,
        display: 'flex',
        flexDirection: 'column',
        gap: 8,
      }}
      role="region"
      aria-label="Notifications"
    >
      {toasts.map(toast => (
        <ToastItem key={toast.id} toast={toast} onDismiss={onDismiss} />
      ))}
    </div>,
    document.body
  );
}

function ToastItem({ toast, onDismiss }: { toast: Toast; onDismiss: (id: string) => void }) {
  // role="alert" — assertive, screen reader darhol e'lon (errors uchun)
  // role="status" — polite, joriy gapni tugatib e'lon (success/info/warning)
  // Hammasiga "alert" qo'yish noto'g'ri: success/info uchun overkill, AT user'larni bezovta qiladi
  const ariaRole = toast.type === 'error' ? 'alert' : 'status';
  
  return (
    <div 
      className={`toast toast-${toast.type}`}
      role={ariaRole}
      style={{
        background: getBackgroundColor(toast.type),
        color: '#fff',
        padding: '12px 16px',
        borderRadius: 4,
        minWidth: 250,
        boxShadow: '0 2px 8px rgba(0,0,0,0.2)',
        display: 'flex',
        alignItems: 'center',
        gap: 12,
      }}
    >
      <span>{getIcon(toast.type)}</span>
      <span style={{ flex: 1 }}>{toast.message}</span>
      <button 
        onClick={() => onDismiss(toast.id)}
        aria-label="Dismiss"
        style={{ background: 'transparent', border: 'none', color: '#fff', cursor: 'pointer' }}
      >
        ×
      </button>
    </div>
  );
}

function getBackgroundColor(type: ToastType): string {
  switch (type) {
    case 'success': return '#22c55e';
    case 'error': return '#ef4444';
    case 'warning': return '#f59e0b';
    case 'info': return '#3b82f6';
  }
}

function getIcon(type: ToastType): string {
  switch (type) {
    case 'success': return '✓';
    case 'error': return '✕';
    case 'warning': return '⚠';
    case 'info': return 'ℹ';
  }
}

// Usage
function App() {
  return (
    <ToastProvider>
      <Dashboard />
    </ToastProvider>
  );
}

function Dashboard() {
  const { toast } = useToast();
  
  const handleSave = async () => {
    try {
      await saveData();
      toast.success('Saved successfully');
    } catch (err) {
      toast.error(`Failed to save: ${(err as Error).message}`);
    }
  };
  
  return (
    <div>
      <button onClick={handleSave}>Save</button>
      <button onClick={() => toast.info('Loading...')}>Show Info</button>
    </div>
  );
}
```

Animation with `framer-motion`:

```tsx
import { motion, AnimatePresence } from 'framer-motion';

function ToastContainer({ toasts, onDismiss }: { toasts: Toast[]; onDismiss: (id: string) => void }) {
  return createPortal(
    <div className="toast-container">
      <AnimatePresence>
        {toasts.map(toast => (
          <motion.div
            key={toast.id}
            initial={{ opacity: 0, x: 100 }}
            animate={{ opacity: 1, x: 0 }}
            exit={{ opacity: 0, x: 100 }}
            transition={{ duration: 0.2 }}
          >
            <ToastItem toast={toast} onDismiss={onDismiss} />
          </motion.div>
        ))}
      </AnimatePresence>
    </div>,
    document.body
  );
}
```

</details>

---

## z-index va Stacking Contexts

### Nazariya

CSS `z-index` va **stacking contexts** Portal'larning eng murakkab CSS aspect'i. Stacking context — DOM element guruhi, z-index hierarchy ichida.

Stacking context yaratuvchi CSS:

- `position: fixed`/`sticky`
- `position: absolute`/`relative` + `z-index: <number>` (auto emas)
- `opacity < 1`
- `transform: <any>` (none emas)
- `filter: <any>` (none emas)
- `will-change: <any>`
- `isolation: isolate`
- `mix-blend-mode: <any>` (normal emas)
- `contain: layout`/`paint`/`strict`/`content`
- `<dialog>` element (top layer)

Muammo:

```tsx
function App() {
  return (
    <div style={{ transform: 'translateZ(0)', zIndex: 1 }}>
      {/* Stacking context — `transform` */}
      <Modal>  {/* z-index: 9999 */}
      </Modal>
    </div>
  );
}
// Modal `z-index: 9999` parent stacking context (transform) ichida
// Boshqa elements `z-index: 100` parent context tashqarisida — Modal yopadi (Modal context tashqarisida ko'rinmaydi)
```

NIMA UCHUN Portal `<body>`'ga render — bu muammoni hal qiladi. `<body>` top-level stacking context. Modal har qanday parent stacking context'dan tashqari bo'ladi.

QANDAY ISHLAYDI:

```tsx
// ❌ Without Portal — Modal trapped in parent stacking context
<div style={{ transform: 'translateZ(0)' }}>  {/* stacking context */}
  <Modal style={{ zIndex: 9999 }} />
  {/* Modal z-index parent context ichida ishlaydi, lekin parent context bir
     butun unit sifatida grandparent'da boshqa stacking context'lar bilan
     solishtiriladi — parent'dan tashqaridagi z-index: 100 bo'lgan div
     Modal'ni yopib qo'yishi mumkin */}
</div>

// ✅ With Portal — Modal rendered to body
<div style={{ transform: 'translateZ(0)' }}>
  {createPortal(<Modal style={{ zIndex: 9999 }} />, document.body)}
  {/* Modal body'da — top-level stacking context, z-index: 9999 global ishlaydi */}
</div>
```

z-index management strategy:

```tsx
// CSS variables for layered z-index
:root {
  --z-base: 0;
  --z-dropdown: 1000;
  --z-tooltip: 1100;
  --z-modal-backdrop: 1200;
  --z-modal: 1300;
  --z-toast: 1400;
  --z-popover: 1500;
}

.modal-backdrop { z-index: var(--z-modal-backdrop); }
.tooltip { z-index: var(--z-tooltip); }
.toast { z-index: var(--z-toast); }
```

Centralized z-index management — har layer aniq bilinadi.

<details>
<summary><strong>Under the Hood</strong></summary>

Browser stacking algorithm (simplified):

```
1. Stacking contexts identify (root, position+z-index, transform, opacity, etc.)
2. Each context has z-index ladder:
   - Negative z-index (background-most)
   - Block elements (default)
   - Floats
   - Inline elements
   - Positioned elements with z-index: auto/0
   - Positioned elements with z-index > 0 (sorted)
3. Render in order, child contexts always above parent's positioned children
```

Problem visualization:

```
DOM:
  body
    └── div.parent (stacking context — transform)
          ├── div.modal (z-index: 9999)  ← faqat parent context ichida amal qiladi
          └── div.sibling (z-index: 1)
    └── div.aside (z-index: 1000)        ← parent context'dan tashqarida — body'ning bevosita bolasi

Visual order (z-index from front to back):
  - div.aside (1000) — body'ning to'g'ridan-to'g'ri bolasi, body context'da
  - div.parent (transform context, default z-index: auto):
    - div.modal (9999 — faqat parent ichida)
    - div.sibling (1 — faqat parent ichida)

Modal NEVER above div.aside — parent stacking context unit sifatida aside'dan past
(parent default z-index: auto, aside z-index: 1000)
```

Portal solution:

```
DOM:
  body
    └── div.parent (transform)
          └── div.sibling
    └── div.aside (z-index: 1000)
    └── div.modal-portal (z-index: 9999)  ← Top-level via Portal

Visual order:
  - div.modal-portal (9999) — Top-level
  - div.aside (1000)
  - div.parent context children
```

Modern alternative — **`<dialog>` element + top layer** (CSS):

```html
<dialog open>
  <h2>Modal in top layer</h2>
</dialog>
```

`<dialog>` element `.showModal()` orqali chaqirilganda **CSS top layer**'ga ko'chiriladi — z-index'siz har stacking context'dan ustun (top layer DOM tree dan ham, paint order dan ham tashqarida — alohida render layer).

**`.showModal()` native xususiyatlari:**
- **Top layer** — barcha stacking context'lardan ustun (`transform`/`opacity`/`filter` parent'lardan ozod), z-index management kerak emas
- **`::backdrop`** — pseudo-element backdrop styling uchun (`dialog::backdrop { background: rgba(0,0,0,0.5); }`)
- **Avtomatik focus** — birinchi focusable element'ga focus qo'yiladi (`autofocus` attribute orqali boshqarish mumkin)
- **Focus trap** — Tab key dialog ichida cycle (Chrome 90+, Firefox 118+, Safari 17+)
- **ESC key** — native handling: `close` event fire qiladi (preventDefault bilan to'sish mumkin; HTML 2024 `closedby` attribute fine-grained nazorat beradi)
- **Stack** — bir nechta `.showModal()` chaqirig'i stack hosil qiladi, oxirgi ochilgan eng tepada
- **`return value`** — `<form method="dialog">` yoki `dialog.close('value')` orqali natija qaytarish

**Portal-based Modal vs `<dialog>` taqqoslash:**

| Xususiyat | Portal Modal | `<dialog>.showModal()` |
|-----------|--------------|------------------------|
| Top layer | ❌ (z-index management) | ✅ Native |
| ESC handling | ❌ Manual listener | ✅ Native |
| Focus trap | ❌ Manual (yoki `inert`) | ✅ Native (modern browser) |
| Backdrop styling | CSS class | `::backdrop` pseudo-element |
| Animation control | ✅ To'liq erkinlik | ⚠️ `display: none` muammosi (workaround: `@starting-style`, `transition-behavior: allow-discrete`) |
| Browser support | R16+ (2017) — universal | Chrome 37+/Firefox 98+/Safari 15.4+ (cross-browser 2022-mart) |
| SSR | Manual handling | Element'ning o'zi SSR-safe |

**Qachon Portal afzal:** Animation flexibility kerak (slide-in drawer, complex transition), eski browser (Firefox <98, Safari <15.4) qo'llab-quvvatlash zarur, yoki nested modal'larning custom stack management kerak. Aks holda `<dialog>` afzal.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

z-index management:

```css
/* Centralized z-index variables */
:root {
  --z-content: 1;
  --z-dropdown: 100;
  --z-sticky-header: 200;
  --z-tooltip: 300;
  --z-modal-backdrop: 400;
  --z-modal: 401;
  --z-toast: 500;
  --z-popover: 600;
  --z-overlay: 700;
}

.modal-backdrop { z-index: var(--z-modal-backdrop); }
.modal { z-index: var(--z-modal); }
.tooltip { z-index: var(--z-tooltip); }
.toast-container { z-index: var(--z-toast); }
.dropdown-popup { z-index: var(--z-dropdown); }
```

Stacking context awareness:

```tsx
// ❌ Anti-pattern — transform on parent breaks z-index
function ParentWithTransform() {
  return (
    <div style={{ transform: 'translateY(0)' }}>
      <Modal />  {/* Modal parent stacking context'da qoladi — z-index: 9999 faqat shu context ichida */}
    </div>
  );
}

// ✅ Use Portal — escape parent stacking
function CorrectPattern() {
  return (
    <div style={{ transform: 'translateY(0)' }}>
      {/* Modal renders to body, no stacking issue */}
      <Modal />
    </div>
  );
}

// ✅ Or avoid creating stacking context
function AvoidContext() {
  return (
    <div style={{ /* no transform, no opacity, no z-index */ }}>
      <Modal />
    </div>
  );
}
```

Modern `<dialog>` element:

```tsx
function NativeModal({ isOpen, onClose, children }: ModalProps) {
  const dialogRef = useRef<HTMLDialogElement>(null);
  
  // useEffect (commit'dan keyin) — DOM mavjud bo'lganda showModal/close chaqirish
  useEffect(() => {
    const dialog = dialogRef.current;
    if (!dialog) return;
    
    if (isOpen && !dialog.open) {
      dialog.showModal();
    } else if (!isOpen && dialog.open) {
      dialog.close();
    }
  }, [isOpen]);
  
  return (
    <dialog 
      ref={dialogRef}
      // ESC key yoki dialog.close() chaqirilganda fire bo'ladi — state sync uchun
      onClose={onClose}
      onClick={(e) => {
        // Backdrop click — dialog element'ning o'zi target (content ichida emas)
        // Eslatma: dialog'da padding bo'lsa, padding zone ham backdrop hisoblanadi.
        // Yaxshiroq: getBoundingClientRect tekshiruvi
        const rect = dialogRef.current?.getBoundingClientRect();
        if (!rect) return;
        const isInDialog = 
          rect.top <= e.clientY && e.clientY <= rect.top + rect.height &&
          rect.left <= e.clientX && e.clientX <= rect.left + rect.width;
        if (!isInDialog) onClose();
      }}
    >
      {children}
    </dialog>
  );
}

// Avzaliklari:
// - Top layer (z-index management yo'q)
// - Native ESC (manual listener kerak emas)  
// - Native focus trap (modern browser'larda)
// - aria-modal="true" implicit
// Browser support: Chrome 37+, Firefox 98+, Safari 15.4+ (cross-browser 2022-mart)
```

`::backdrop` styling:

```css
dialog::backdrop {
  background: rgba(0, 0, 0, 0.5);
  backdrop-filter: blur(4px);
}

/* Open animation — @starting-style (Chrome 117+, Firefox 129+, Safari 17.5+) */
dialog {
  opacity: 0;
  transform: scale(0.95);
  transition: opacity 0.2s, transform 0.2s, overlay 0.2s allow-discrete, display 0.2s allow-discrete;
}

dialog[open] {
  opacity: 1;
  transform: scale(1);
}

@starting-style {
  dialog[open] {
    opacity: 0;
    transform: scale(0.95);
  }
}
```

`@starting-style` + `transition-behavior: allow-discrete` — `<dialog>` ochilish animation muammosini hal qiladi (eski browser'larda `display: none` transition'lar bekor qilardi).

</details>

---

## Focus Management va Focus Trap

### Nazariya

Focus management — accessibility'ning fundamental aspect'i. Modal/Dialog/Popover ochilganda:

1. **Initial focus** — birinchi focusable element'ga ko'chirish.
2. **Focus trap** — Tab key Modal ichida cycle qiladi, tashqariga chiqmaydi.
3. **Focus return** — yopilganda triggering element'ga focus qaytarish.

NIMA UCHUN focus trap:

- Keyboard user'lar Modal ichida navigate qiladi.
- Tab key Modal tashqariga chiqsa — context lost (foydalanuvchi qayerga focus tushganini bilmaydi).
- ARIA spec talab qiladi (`aria-modal="true"` bilan).

> **Modern pattern (afzal):** Manual Tab handler o'rniga **HTML `inert` attribute** background element'ga qo'yish — focus trap avtomatik (Chrome 102+/Firefox 112+/Safari 15.5+). Yoki **`<dialog>.showModal()`** — focus trap native (modern browser'larda spec'ga muvofiq). Manual Tab cycle pattern faqat eski browser fallback yoki `inert` mavjud bo'lmagan kontekst uchun. Manual pattern'ning ma'lum kamchiligi: focus modal tashqarisida bo'lsa (masalan, browser chrome'da yoki dev tools'da), Tab key trap'ga yetib bormaydi — bunday holatda `inert` background ishonchli yechim.

QANDAY ISHLAYDI:

```tsx
function FocusTrap({ children }: { children: React.ReactNode }) {
  const containerRef = useRef<HTMLDivElement>(null);
  
  useEffect(() => {
    const focusableSelectors = 
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])';
    const focusables = containerRef.current?.querySelectorAll<HTMLElement>(focusableSelectors);
    
    if (!focusables?.length) return;
    
    const first = focusables[0];
    const last = focusables[focusables.length - 1];
    
    // Initial focus
    first.focus();
    
    const handleTab = (e: KeyboardEvent) => {
      if (e.key !== 'Tab') return;
      
      if (e.shiftKey && document.activeElement === first) {
        e.preventDefault();
        last.focus();  // Backward to last
      } else if (!e.shiftKey && document.activeElement === last) {
        e.preventDefault();
        first.focus();  // Forward to first
      }
    };
    
    document.addEventListener('keydown', handleTab);
    return () => document.removeEventListener('keydown', handleTab);
  }, []);
  
  return <div ref={containerRef}>{children}</div>;
}
```

Focus return pattern:

```tsx
function Modal({ isOpen, onClose, children }: ModalProps) {
  const previousFocusRef = useRef<HTMLElement | null>(null);
  
  useEffect(() => {
    if (isOpen) {
      previousFocusRef.current = document.activeElement as HTMLElement;
    } else {
      previousFocusRef.current?.focus();
    }
  }, [isOpen]);
  
  // ... modal render
}
```

HTML `inert` attribute — alternative pattern. R19'da React boolean attribute sifatida proper render qiladi (R18'gacha string serialization edi):

```tsx
// inert — child elements'larni ignore qiladi (focus + events)
<>
  <div inert={isModalOpen}>
    <App />  {/* All app inert when modal open */}
  </div>
  <Modal isOpen={isModalOpen} />
</>
```

**Browser support:** Chrome 102+, Firefox 112+, Safari 15.5+ (HTML standard, React tomonidan kiritilmagan). Background ichidagi focusable element'lar Tab key bilan focus olmaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

Focusable selectors:

```javascript
const FOCUSABLE_SELECTORS = [
  'button:not([disabled])',
  '[href]',
  'input:not([disabled])',
  'select:not([disabled])',
  'textarea:not([disabled])',
  '[tabindex]:not([tabindex="-1"]):not([disabled])',
  'audio[controls]',
  'video[controls]',
  '[contenteditable]:not([contenteditable="false"])',
].join(', ');
```

Edge cases:
- `disabled` attribute — exclude
- `tabindex="-1"` — exclude (programmatic focus only)
- `[hidden]` — exclude
- CSS `display: none` — exclude (already not focusable)

`document.activeElement` — joriy focused element. `null` agar nothing focused.

`element.focus({ preventScroll: true })` — focus without scroll (avoid jumping).

R18+ Strict Mode 2x effect cycle — focus trap ikki marta setup. Initial focus 2x. UX'da farq yo'q (deterministic), lekin metric'larda visible.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Production focus trap hook:

```tsx
import { useEffect, RefObject } from 'react';

export function useFocusTrap(
  containerRef: RefObject<HTMLElement>,
  active: boolean
) {
  useEffect(() => {
    if (!active || !containerRef.current) return;
    
    const container = containerRef.current;
    const focusableSelectors = 
      'button:not([disabled]), [href], input:not([disabled]), select:not([disabled]), textarea:not([disabled]), [tabindex]:not([tabindex="-1"]):not([disabled])';
    
    function getFocusables(): HTMLElement[] {
      return Array.from(container.querySelectorAll<HTMLElement>(focusableSelectors));
    }
    
    // Initial focus
    const focusables = getFocusables();
    focusables[0]?.focus();
    
    // Tab key handler
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key !== 'Tab') return;
      
      const focusables = getFocusables();
      if (focusables.length === 0) return;
      
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
    
    document.addEventListener('keydown', handleKeyDown);
    return () => document.removeEventListener('keydown', handleKeyDown);
  }, [active, containerRef]);
}

// useFocusReturn — focus restore on unmount
export function useFocusReturn(active: boolean) {
  useEffect(() => {
    if (!active) return;
    
    const previousFocus = document.activeElement as HTMLElement;
    
    return () => {
      previousFocus?.focus();
    };
  }, [active]);
}

// Combined Modal hooks
function AccessibleModal({ isOpen, onClose, children }: ModalProps) {
  const modalRef = useRef<HTMLDivElement>(null);
  
  // Tartib muhim: useFocusReturn birinchi — `document.activeElement` (trigger button)
  // hali modal element'iga ko'chmagan paytda capture qiladi. Keyin useFocusTrap focus'ni
  // modal ichkarisiga olib o'tadi. Tartib teskari bo'lsa, previousFocus modal'ning ichki
  // elementiga ishora qiladi (bug).
  useFocusReturn(isOpen);
  useFocusTrap(modalRef, isOpen);
  
  if (!isOpen) return null;
  
  return createPortal(
    <div className="modal-backdrop" onClick={onClose}>
      <div 
        ref={modalRef}
        className="modal"
        role="dialog"
        aria-modal="true"
        onClick={(e) => e.stopPropagation()}
      >
        {children}
      </div>
    </div>,
    document.body
  );
}
```

HTML `inert` attribute (R19+ React boolean prop support):

```tsx
function App() {
  const [isModalOpen, setIsModalOpen] = useState(false);
  
  return (
    <>
      <div inert={isModalOpen}>
        {/* Background app — inert when modal open. R19'da boolean prop ishlaydi. */}
        <Header />
        <MainContent />
        <Footer />
      </div>
      
      <Modal isOpen={isModalOpen} onClose={() => setIsModalOpen(false)} />
    </>
  );
}
// inert background — focus trap automatic, no manual Tab handling
```

`inert` browser-native — focus + click events ignore. Modal'larda focus trap kerak emas (browser built-in).

`react-focus-lock` library:

```tsx
import FocusLock from 'react-focus-lock';

function Modal({ isOpen, children }: ModalProps) {
  if (!isOpen) return null;
  
  return createPortal(
    <FocusLock returnFocus>
      <div className="modal-backdrop">
        <div className="modal" role="dialog" aria-modal="true">
          {children}
        </div>
      </div>
    </FocusLock>,
    document.body
  );
}
```

Production library — battle-tested, edge cases handled (iframe, shadow DOM).

</details>

---

## SSR Considerations

### Nazariya

Portal'lar **SSR (Server-Side Rendering)** bilan ishlash murakkab — server'da `document` global mavjud emas. Tipik pattern `createPortal(children, document.body)` server'da `ReferenceError: document is not defined` beradi (xato `document.body` access'idan, `createPortal` o'zidan emas — funksiya target argument'sini evaluate qilolmaydi).

```tsx
// ❌ SSR'da error: ReferenceError: document is not defined
function Modal() {
  return createPortal(<div>Modal</div>, document.body);
}
```

QANDAY ISHLAYDI: yechim — **client-only render**:

**Pattern 1: `mounted` state**

```tsx
function ClientOnlyPortal({ children, target }: { children: React.ReactNode; target: HTMLElement | null }) {
  const [mounted, setMounted] = useState(false);
  
  useEffect(() => {
    setMounted(true);
  }, []);
  
  if (!mounted || !target) return null;
  return createPortal(children, target);
}
```

`useEffect` faqat client'da chaqiriladi (server skip). Initial render `null`, post-mount Portal render.

**Pattern 2: `typeof window` check**

```tsx
function Portal({ children, target }: { children: React.ReactNode; target?: HTMLElement }) {
  if (typeof window === 'undefined') return null;
  
  const container = target ?? document.body;
  return createPortal(children, container);
}
```

`typeof window === 'undefined'` — SSR check. Server'da skip, client'da render.

**Pattern 3: Next.js dynamic import**

```tsx
import dynamic from 'next/dynamic';

const Modal = dynamic(() => import('./Modal'), { ssr: false });

// Modal komponent server'da render qilinmaydi — faqat client
```

NIMA UCHUN bu cheklov: SSR'da HTML server'da generate qilinadi, lekin `document.body` mavjud emas (DOM API yo'q). Portal client-only.

**Hydration considerations** — Portal initial server HTML'da yo'q, client-only mount paytida qo'shiladi:

```html
<!-- Server HTML (no portal) -->
<body>
  <div id="root">
    <main>App content</main>
  </div>
</body>

<!-- After hydration + client-only Portal mount -->
<body>
  <div id="root">
    <main>App content</main>
  </div>
  <div class="modal">  ← Portal'd post-hydration
    Modal content
  </div>
</body>
```

Hydration mismatch yo'q (server Portal'siz, client Portal qo'shadi).

R19 Document Metadata API (cross-ref [`37-react-19-document-apis.md`](37-react-19-document-apis.md)) — `<title>`/`<meta>`/`<link>` Portal'siz `<head>`'ga hoist. Bu Portal use case'larining bir qismini almashtiradi.

<details>
<summary><strong>Under the Hood</strong></summary>

Server vs Client environment:

| Environment | `window` | `document` | `createPortal` |
|-------------|----------|------------|-----------------|
| Browser (client) | ✅ Mavjud | ✅ Mavjud | ✅ Ishlaydi |
| Node.js (SSR) | ❌ Undefined | ❌ Undefined | ❌ Error |
| React Server Components | ❌ Undefined | ❌ Undefined | ❌ Error (use client kerak) |

Next.js / Remix / Astro — SSR frameworks. Portal client-only komponent'larda ishlatilishi kerak.

`'use client'` directive (Next.js 13+ App Router):

```tsx
'use client';

import { createPortal } from 'react-dom';
import { useState, useEffect } from 'react';

export function Modal({ children }: { children: React.ReactNode }) {
  const [mounted, setMounted] = useState(false);
  
  useEffect(() => {
    setMounted(true);
  }, []);
  
  if (!mounted) return null;
  return createPortal(children, document.body);
}
```

`'use client'` directive — komponent client'da ishlaydi, server'da serialize qilinmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

SSR-safe Portal hook:

```tsx
import { useEffect, useState } from 'react';
import { createPortal } from 'react-dom';

export function useSSRSafePortal() {
  const [mounted, setMounted] = useState(false);
  
  useEffect(() => {
    setMounted(true);
  }, []);
  
  return {
    Portal: ({ children, target }: { children: React.ReactNode; target?: HTMLElement }) => {
      if (!mounted) return null;
      return createPortal(children, target ?? document.body);
    },
  };
}

// Usage
function Modal({ isOpen, children }: { isOpen: boolean; children: React.ReactNode }) {
  const { Portal } = useSSRSafePortal();
  
  if (!isOpen) return null;
  
  return (
    <Portal>
      <div className="modal">{children}</div>
    </Portal>
  );
}
```

Next.js dynamic import:

```tsx
// app/components/Modal.tsx
'use client';

import { createPortal } from 'react-dom';

export default function Modal({ children }: { children: React.ReactNode }) {
  return createPortal(<div className="modal">{children}</div>, document.body);
}

// app/page.tsx
import dynamic from 'next/dynamic';

const Modal = dynamic(() => import('./components/Modal'), {
  ssr: false,
  loading: () => null,
});

export default function Page() {
  return (
    <main>
      <h1>Page</h1>
      <Modal>Modal content</Modal>
    </main>
  );
}
```

`{ ssr: false }` — komponent server'da render qilinmaydi. Bundle'da alohida chunk (lazy-loaded).

</details>

---

## Portal Cleanup va Lifecycle

### Nazariya

Portal lifecycle — komponent'ning normal lifecycle bilan integrated:

1. **Mount** — Portal element create, container'ga `appendChild`.
2. **Update** — children o'zgarsa, Reconciler diff Portal children.
3. **Unmount** — Portal element container'dan `removeChild`.

NIMA UCHUN cleanup muhim:

1. **Memory leak** — DOM element memory'da qoladi agar properly unmount qilinmasa.
2. **Event listeners** — Portal ichidagi listener'lar saqlanadi.
3. **Dynamic containers** — `useEffect` orqali container yaratilsa, cleanup container'ni o'chirishi kerak.

```tsx
function DynamicPortal({ children }: { children: React.ReactNode }) {
  const [container, setContainer] = useState<HTMLElement | null>(null);
  
  useEffect(() => {
    const div = document.createElement('div');
    div.className = 'dynamic-portal';
    document.body.appendChild(div);
    setContainer(div);
    
    return () => {
      // Cleanup — remove container from DOM
      if (div.parentElement) {
        div.parentElement.removeChild(div);
      }
    };
  }, []);
  
  if (!container) return null;
  return createPortal(children, container);
}
```

Cleanup paytida:

1. React Reconciler Portal Fiber'ini destroy qiladi.
2. Children'lar container'dan automatically unmount.
3. `useEffect` cleanup function chaqiriladi (custom cleanup logic).

QANDAY ISHLAYDI:

```
Component mounts
   │
   ├─ useEffect setup: create container + appendChild
   ├─ setContainer(div)
   └─ Re-render with container
       │
       └─ createPortal(children, container)
           ├─ Create Portal Fiber
           ├─ Reconcile children
           └─ DOM commit children to container

Component unmounts
   │
   ├─ useEffect cleanup: remove container
   │   └─ document.body.removeChild(div)
   │       └─ Children automatically removed
   └─ Portal Fiber destroyed
```

<details>
<summary><strong>Under the Hood</strong></summary>

React Reconciler unmount:

```javascript
// react-reconciler/src/ReactFiberCommitWork.js (simplified)
function commitDeletion(fiber) {
  if (fiber.tag === HostPortal) {
    const container = fiber.stateNode.containerInfo;
    
    // Remove all children from container
    fiber.children?.forEach(child => {
      container.removeChild(child.stateNode);
    });
    
    // Cleanup Portal Fiber
    fiber.stateNode = null;
  }
}
```

Reconciler children'larni container'dan remove qiladi. Custom container — manual cleanup `useEffect` da.

Memory profiler trace:

```
Before mount:
  - DOM nodes: 100
  - React Fibers: 50

After mount Portal:
  - DOM nodes: 105 (+5 portal children)
  - React Fibers: 51 (+1 portal Fiber)

After unmount:
  - DOM nodes: 100 (back to original)
  - React Fibers: 50
```

Memory cleanup — automatic (React handles).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Portal with cleanup:

```tsx
import React, { useState, useEffect } from 'react';
import { createPortal } from 'react-dom';

interface PortalProps {
  children: React.ReactNode;
  className?: string;
}

export function Portal({ children, className = 'portal' }: PortalProps) {
  const [container, setContainer] = useState<HTMLElement | null>(null);
  
  useEffect(() => {
    const div = document.createElement('div');
    div.className = className;
    document.body.appendChild(div);
    setContainer(div);
    
    return () => {
      if (div.parentElement) {
        div.parentElement.removeChild(div);
      }
    };
  }, [className]);
  
  if (!container) return null;
  return createPortal(children, container);
}

// Usage
function App() {
  const [showModal, setShowModal] = useState(false);
  
  return (
    <>
      <button onClick={() => setShowModal(true)}>Open Modal</button>
      
      {showModal && (
        <Portal className="modal-portal">
          <div className="modal">
            <button onClick={() => setShowModal(false)}>Close</button>
          </div>
        </Portal>
      )}
    </>
  );
}
```

Reusable shared container:

```tsx
// Shared container — multiple modals
let sharedContainer: HTMLElement | null = null;
let usageCount = 0;

function getSharedContainer(): HTMLElement {
  if (!sharedContainer) {
    sharedContainer = document.createElement('div');
    sharedContainer.className = 'modals-container';
    document.body.appendChild(sharedContainer);
  }
  usageCount++;
  return sharedContainer;
}

function releaseSharedContainer() {
  usageCount--;
  if (usageCount === 0 && sharedContainer?.parentElement) {
    sharedContainer.parentElement.removeChild(sharedContainer);
    sharedContainer = null;
  }
}

export function SharedPortal({ children }: { children: React.ReactNode }) {
  const [container, setContainer] = useState<HTMLElement | null>(null);
  
  useEffect(() => {
    setContainer(getSharedContainer());
    return () => releaseSharedContainer();
  }, []);
  
  if (!container) return null;
  return createPortal(children, container);
}
```

Counter-based cleanup — bir vaqtning o'zida bir nechta modal ochilsa, container saqlanadi. Hammasi yopilganda — cleanup.

</details>

---

## Edge Cases va Gotchas

### Gotcha 1: `event.target` Portal'da DOM tree bo'ylab

`event.target` — DOM element. Portal'd element'da click qilinsa, `event.target` Portal element (DOM joyida — body'da). React Fiber tree esa parent komponent'iga ulangan.

```tsx
function App() {
  const handleClick = (e: React.MouseEvent) => {
    console.log('Target:', e.target);              // DOM element (Portal'd)
    console.log('CurrentTarget:', e.currentTarget); // App's div
  };
  
  return (
    <div onClick={handleClick}>
      <Modal>
        <button>Click me</button>  {/* target: button, currentTarget: App div */}
      </Modal>
    </div>
  );
}
```

`e.target` DOM-based, `e.currentTarget` React tree-based handler element.

### Gotcha 2: `ref` Portal element'ga DOM joyida access

`ref` Portal'd element'ga DOM joyida access beradi:

```tsx
function Modal({ children }: { children: React.ReactNode }) {
  const modalRef = useRef<HTMLDivElement>(null);
  
  useEffect(() => {
    if (modalRef.current) {
      console.log('Parent:', modalRef.current.parentElement);
      // DOM parent — body (Portal target), not React parent
    }
  }, []);
  
  return createPortal(<div ref={modalRef}>{children}</div>, document.body);
}
```

Ref DOM tree'ni reflect qiladi (Portal target). React tree parent'ga access yo'q.

### Gotcha 3: CSS `:hover` parent selector ishlamaydi Portal'd element'da

```css
.parent:hover .modal {
  /* ❌ Won't work — modal Portal'd to body */
  background: red;
}

/* ✅ Use JavaScript state */
```

CSS selectors DOM tree-based. Portal element parent CSS rules'dan tashqarida.

### Gotcha 4: Portal `key` Reconciliation siblings

Multiple Portals bir parent'da — React Reconciliation `key` ishlatadi:

```tsx
function NotificationStack({ messages }: { messages: string[] }) {
  return (
    <>
      {messages.map((msg, i) => 
        createPortal(
          <div className="notification">{msg}</div>,
          document.body,
          // ❌ Without key — siblings re-mount on order change
          // ✅ With key:
          `notif-${i}`
        )
      )}
    </>
  );
}
```

`key` Reconciliation efficiency va state preservation.

### Gotcha 5: Nested Portal'lar

```tsx
// Portal inside Portal — works
function App() {
  return (
    <Modal>
      <Tooltip>Nested portal</Tooltip>
    </Modal>
  );
}

// Both Modal va Tooltip body'ga render
// React tree: App > Modal > Tooltip
// DOM tree: body > Modal, body > Tooltip
```

Nested Portal'lar OK. Lekin React tree'da nested, DOM tree'da flat.

---

## Common Mistakes

### ❌ Xato 1: SSR'da `document.body` access

```tsx
// ❌ Server-side error
function Modal() {
  return createPortal(<div>Modal</div>, document.body);
}

// ✅ Client-only render
function Modal() {
  const [mounted, setMounted] = useState(false);
  
  useEffect(() => {
    setMounted(true);
  }, []);
  
  if (!mounted) return null;
  return createPortal(<div>Modal</div>, document.body);
}
```

### ❌ Xato 2: Stop propagation'siz backdrop click

```tsx
// ❌ Click inside Modal closes Modal
function Modal({ onClose, children }: ModalProps) {
  return createPortal(
    <div onClick={onClose}>  {/* Backdrop click — close */}
      <div>{children}</div>  {/* Click here also fires onClose (bubbles) */}
    </div>,
    document.body
  );
}

// ✅ Stop propagation on Modal content
function Modal({ onClose, children }: ModalProps) {
  return createPortal(
    <div onClick={onClose}>
      <div onClick={(e) => e.stopPropagation()}>  {/* ✅ Block bubbling */}
        {children}
      </div>
    </div>,
    document.body
  );
}
```

### ❌ Xato 3: Cleanup yo'q dynamic container

```tsx
// ❌ Memory leak — container DOM'da qoladi
function BadPortal({ children }: { children: React.ReactNode }) {
  const div = document.createElement('div');
  document.body.appendChild(div);
  return createPortal(children, div);
}

// ✅ useEffect with cleanup
function GoodPortal({ children }: { children: React.ReactNode }) {
  const [container, setContainer] = useState<HTMLElement | null>(null);
  
  useEffect(() => {
    const div = document.createElement('div');
    document.body.appendChild(div);
    setContainer(div);
    
    return () => {
      if (div.parentElement) div.parentElement.removeChild(div);
    };
  }, []);
  
  if (!container) return null;
  return createPortal(children, container);
}
```

### ❌ Xato 4: Focus management yo'q

```tsx
// ❌ Modal opens — focus stays on triggering button (or worse, no focus)
function BadModal({ isOpen, children }: ModalProps) {
  if (!isOpen) return null;
  return createPortal(<div>{children}</div>, document.body);
}

// ✅ Initial focus + return
function GoodModal({ isOpen, onClose, children }: ModalProps) {
  const modalRef = useRef<HTMLDivElement>(null);
  const previousFocus = useRef<HTMLElement | null>(null);
  
  useEffect(() => {
    if (isOpen) {
      previousFocus.current = document.activeElement as HTMLElement;
      modalRef.current?.querySelector<HTMLElement>('button, [href]')?.focus();
    } else {
      previousFocus.current?.focus();
    }
  }, [isOpen]);
  
  if (!isOpen) return null;
  return createPortal(
    <div ref={modalRef}>{children}</div>,
    document.body
  );
}
```

### ❌ Xato 5: z-index parent stacking context

```css
/* ❌ Modal trapped in parent context */
.parent {
  transform: translateZ(0);  /* Creates stacking context */
}
.modal {
  z-index: 9999;  /* Capped at parent level */
}
```

```tsx
// ✅ Portal escapes parent context
function Modal() {
  return createPortal(<div className="modal">Modal</div>, document.body);
  // Modal in body — top-level stacking
}
```

---

## Amaliy Mashqlar

### Mashq 1: Sodda Portal Hook (Oson)

`usePortal(targetId)` hook yarating — target element ID bo'yicha Portal yaratadi. Target mavjud bo'lmasa — yangi yaratadi va cleanup'da olib tashlaydi.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import React, { useState, useEffect } from 'react';
import { createPortal } from 'react-dom';

export function usePortal(targetId: string = 'portal-root'): {
  Portal: (props: { children: React.ReactNode }) => React.ReactElement | null;
  isReady: boolean;
} {
  const [container, setContainer] = useState<HTMLElement | null>(null);
  
  useEffect(() => {
    let target = document.getElementById(targetId);
    let createdNew = false;
    
    if (!target) {
      target = document.createElement('div');
      target.id = targetId;
      document.body.appendChild(target);
      createdNew = true;
    }
    
    setContainer(target);
    
    return () => {
      if (createdNew && target?.parentElement) {
        target.parentElement.removeChild(target);
      }
    };
  }, [targetId]);
  
  const Portal = React.useMemo(
    () => ({ children }: { children: React.ReactNode }) => {
      if (!container) return null;
      return createPortal(children, container);
    },
    [container]
  );
  
  return { Portal, isReady: container !== null };
}

// Usage
function ModalExample() {
  const { Portal, isReady } = usePortal('modal-root');
  const [isOpen, setIsOpen] = useState(false);
  
  return (
    <>
      <button onClick={() => setIsOpen(true)}>Open</button>
      {isReady && isOpen && (
        <Portal>
          <div className="modal">
            <button onClick={() => setIsOpen(false)}>Close</button>
          </div>
        </Portal>
      )}
    </>
  );
}
```

**Tushuntirish:**
- `useEffect` orqali target dynamic yaratish.
- `createdNew` flag — yangi yaratilgan bo'lsa cleanup'da olib tashlash.
- Component existing target'ni saqlaydi (boshqa Portal ishlatishi mumkin).

</details>

### Mashq 2: Sodda Modal (Oson)

`<Modal isOpen onClose>` yarating — Portal'dan body'ga render, backdrop click + Escape key close, ARIA roles.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import React, { useEffect } from 'react';
import { createPortal } from 'react-dom';

interface ModalProps {
  isOpen: boolean;
  onClose: () => void;
  title?: string;
  children: React.ReactNode;
}

export function Modal({ isOpen, onClose, title, children }: ModalProps) {
  useEffect(() => {
    if (!isOpen) return;
    
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onClose();
    };
    
    document.addEventListener('keydown', handleKeyDown);
    return () => document.removeEventListener('keydown', handleKeyDown);
  }, [isOpen, onClose]);
  
  if (!isOpen) return null;
  
  return createPortal(
    <div 
      className="modal-backdrop"
      onClick={onClose}
      role="presentation"
    >
      <div 
        className="modal"
        role="dialog"
        aria-modal="true"
        aria-labelledby={title ? "modal-title" : undefined}
        onClick={(e) => e.stopPropagation()}
      >
        {title && <h2 id="modal-title">{title}</h2>}
        <div className="modal-body">{children}</div>
        <button onClick={onClose} aria-label="Close modal">×</button>
      </div>
    </div>,
    document.body
  );
}

// Usage
function App() {
  const [isOpen, setIsOpen] = useState(false);
  
  return (
    <>
      <button onClick={() => setIsOpen(true)}>Open Modal</button>
      <Modal isOpen={isOpen} onClose={() => setIsOpen(false)} title="Confirmation">
        <p>Are you sure?</p>
        <button onClick={() => setIsOpen(false)}>Cancel</button>
        <button onClick={() => setIsOpen(false)}>Confirm</button>
      </Modal>
    </>
  );
}
```

</details>

### Mashq 3: Tooltip (O'rta)

`<Tooltip content placement>` yarating — hover/focus paytida ko'rinadi, position calculate, viewport-aware (auto-flip if no space).

<details>
<summary><strong>Javob</strong></summary>

```tsx
import React, { useState, useRef, useLayoutEffect, useEffect, useId } from 'react';
import { createPortal } from 'react-dom';

interface TooltipProps {
  content: React.ReactNode;
  placement?: 'top' | 'bottom' | 'left' | 'right';
  children: React.ReactElement;
}

export function Tooltip({ content, placement = 'top', children }: TooltipProps) {
  const [isVisible, setIsVisible] = useState(false);
  const [position, setPosition] = useState({ top: 0, left: 0, actualPlacement: placement });
  // `tick` — resize/scroll paytida qayta calculate'ni majburlash uchun counter.
  // `setIsVisible(true)` ishlamaydi: React same-value setState'da bailout qiladi
  // va useLayoutEffect deps array o'zgarmagani uchun qayta ishga tushmaydi.
  const [tick, setTick] = useState(0);
  const triggerRef = useRef<HTMLElement | null>(null);
  const tooltipRef = useRef<HTMLDivElement>(null);
  const tooltipId = useId();
  
  useLayoutEffect(() => {
    if (!isVisible || !triggerRef.current || !tooltipRef.current) return;
    
    const triggerRect = triggerRef.current.getBoundingClientRect();
    const tooltipRect = tooltipRef.current.getBoundingClientRect();
    const padding = 8;
    
    let top = 0, left = 0;
    let actualPlacement = placement;
    
    function computePosition(p: typeof placement) {
      switch (p) {
        case 'top':
          return {
            top: triggerRect.top - tooltipRect.height - padding,
            left: triggerRect.left + triggerRect.width / 2 - tooltipRect.width / 2,
          };
        case 'bottom':
          return {
            top: triggerRect.bottom + padding,
            left: triggerRect.left + triggerRect.width / 2 - tooltipRect.width / 2,
          };
        case 'left':
          return {
            top: triggerRect.top + triggerRect.height / 2 - tooltipRect.height / 2,
            left: triggerRect.left - tooltipRect.width - padding,
          };
        case 'right':
          return {
            top: triggerRect.top + triggerRect.height / 2 - tooltipRect.height / 2,
            left: triggerRect.right + padding,
          };
      }
    }
    
    let pos = computePosition(placement);
    
    // Auto-flip if outside viewport
    if (pos.top < padding && placement === 'top') {
      actualPlacement = 'bottom';
      pos = computePosition('bottom');
    } else if (pos.top + tooltipRect.height > window.innerHeight - padding && placement === 'bottom') {
      actualPlacement = 'top';
      pos = computePosition('top');
    } else if (pos.left < padding && placement === 'left') {
      actualPlacement = 'right';
      pos = computePosition('right');
    } else if (pos.left + tooltipRect.width > window.innerWidth - padding && placement === 'right') {
      actualPlacement = 'left';
      pos = computePosition('left');
    }
    
    // Adjust horizontal overflow
    if (pos.left < padding) pos.left = padding;
    if (pos.left + tooltipRect.width > window.innerWidth - padding) {
      pos.left = window.innerWidth - tooltipRect.width - padding;
    }
    
    setPosition({ ...pos, actualPlacement });
  }, [isVisible, placement, tick]);
  
  useEffect(() => {
    if (!isVisible) return;
    
    const handler = () => setTick(t => t + 1);
    window.addEventListener('resize', handler);
    window.addEventListener('scroll', handler, true);
    
    return () => {
      window.removeEventListener('resize', handler);
      window.removeEventListener('scroll', handler, true);
    };
  }, [isVisible]);
  
  const trigger = React.cloneElement(children, {
    ref: triggerRef,
    onMouseEnter: () => setIsVisible(true),
    onMouseLeave: () => setIsVisible(false),
    onFocus: () => setIsVisible(true),
    onBlur: () => setIsVisible(false),
    'aria-describedby': isVisible ? tooltipId : undefined,
  });
  
  return (
    <>
      {trigger}
      {isVisible && createPortal(
        <div 
          ref={tooltipRef}
          id={tooltipId}
          className={`tooltip tooltip-${position.actualPlacement}`}
          role="tooltip"
          style={{
            position: 'fixed',
            top: position.top,
            left: position.left,
            zIndex: 1000,
          }}
        >
          {content}
        </div>,
        document.body
      )}
    </>
  );
}

// Usage
<Tooltip content="Click to delete" placement="top">
  <button>Delete</button>
</Tooltip>
```

</details>

### Mashq 4: Toast Notification System (O'rta)

`<ToastProvider>` + `useToast` yarating — `toast.success/error/info`, queue management, auto-dismiss, animation.

<details>
<summary><strong>Javob</strong></summary>

Yuqorida "Real-World: Toast / Notification System" section'ida to'liq berilgan. Asosiy elementlar:

```tsx
import React, { createContext, useContext, useState, useCallback } from 'react';
import { createPortal } from 'react-dom';

type ToastType = 'success' | 'error' | 'info' | 'warning';

interface Toast {
  id: string;
  type: ToastType;
  message: string;
  duration: number;
}

interface ToastContextValue {
  toast: {
    success: (message: string, duration?: number) => void;
    error: (message: string, duration?: number) => void;
    info: (message: string, duration?: number) => void;
    warning: (message: string, duration?: number) => void;
  };
  dismiss: (id: string) => void;
}

const ToastContext = createContext<ToastContextValue | null>(null);

export function useToast() {
  const ctx = useContext(ToastContext);
  if (!ctx) throw new Error('useToast must be used inside <ToastProvider>');
  return ctx;
}

export function ToastProvider({ children }: { children: React.ReactNode }) {
  const [toasts, setToasts] = useState<Toast[]>([]);
  
  const addToast = useCallback((type: ToastType, message: string, duration: number = 4000) => {
    const id = `toast-${Date.now()}-${Math.random().toString(36).slice(2)}`;
    setToasts(prev => [...prev, { id, type, message, duration }]);
    
    if (duration > 0) {
      setTimeout(() => {
        setToasts(prev => prev.filter(t => t.id !== id));
      }, duration);
    }
  }, []);
  
  const dismiss = useCallback((id: string) => {
    setToasts(prev => prev.filter(t => t.id !== id));
  }, []);
  
  const value: ToastContextValue = {
    toast: {
      success: (msg, dur) => addToast('success', msg, dur),
      error: (msg, dur) => addToast('error', msg, dur),
      info: (msg, dur) => addToast('info', msg, dur),
      warning: (msg, dur) => addToast('warning', msg, dur),
    },
    dismiss,
  };
  
  return (
    <ToastContext.Provider value={value}>
      {children}
      {toasts.length > 0 && createPortal(
        <div 
          className="toast-container"
          role="region"
          aria-live="polite"
          style={{ position: 'fixed', top: 16, right: 16, zIndex: 9999 }}
        >
          {toasts.map(t => (
            <div 
              key={t.id} 
              className={`toast toast-${t.type}`} 
              role={t.type === 'error' ? 'alert' : 'status'}
            >
              {t.message}
              <button onClick={() => dismiss(t.id)} aria-label="Dismiss">×</button>
            </div>
          ))}
        </div>,
        document.body
      )}
    </ToastContext.Provider>
  );
}

// Usage
function App() {
  return (
    <ToastProvider>
      <Dashboard />
    </ToastProvider>
  );
}

function Dashboard() {
  const { toast } = useToast();
  return (
    <div>
      <button onClick={() => toast.success('Saved!')}>Save</button>
      <button onClick={() => toast.error('Failed')}>Error</button>
    </div>
  );
}
```

</details>

### Mashq 5: Production Drawer Component (Qiyin)

`<Drawer>` (slide-in side panel) yarating — Portal, focus trap, body scroll lock, animation (CSS transitions), placement (left/right/top/bottom), ARIA.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import React, { useEffect, useRef, useId } from 'react';
import { createPortal } from 'react-dom';

interface DrawerProps {
  isOpen: boolean;
  onClose: () => void;
  placement?: 'left' | 'right' | 'top' | 'bottom';
  size?: number | string;
  title?: string;
  children: React.ReactNode;
}

export function Drawer({ 
  isOpen, 
  onClose, 
  placement = 'right', 
  size = 400, 
  title,
  children 
}: DrawerProps) {
  const drawerRef = useRef<HTMLDivElement>(null);
  const previousFocusRef = useRef<HTMLElement | null>(null);
  const titleId = useId();
  
  useEffect(() => {
    if (!isOpen) return;
    
    // Save current focus
    previousFocusRef.current = document.activeElement as HTMLElement;
    
    // Body scroll lock
    const originalOverflow = document.body.style.overflow;
    document.body.style.overflow = 'hidden';
    
    // Initial focus
    const focusables = drawerRef.current?.querySelectorAll<HTMLElement>(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    focusables?.[0]?.focus();
    
    // Escape key
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === 'Escape') {
        onClose();
        return;
      }
      
      // Focus trap (Tab key)
      if (e.key === 'Tab' && focusables?.length) {
        const first = focusables[0];
        const last = focusables[focusables.length - 1];
        
        if (e.shiftKey && document.activeElement === first) {
          e.preventDefault();
          last.focus();
        } else if (!e.shiftKey && document.activeElement === last) {
          e.preventDefault();
          first.focus();
        }
      }
    };
    
    document.addEventListener('keydown', handleKeyDown);
    
    return () => {
      document.body.style.overflow = originalOverflow;
      document.removeEventListener('keydown', handleKeyDown);
      previousFocusRef.current?.focus();
    };
  }, [isOpen, onClose]);
  
  if (!isOpen) return null;
  
  const sizeStyle = typeof size === 'number' ? `${size}px` : size;
  
  const placementStyles: Record<typeof placement, React.CSSProperties> = {
    left: { 
      top: 0, 
      left: 0, 
      bottom: 0, 
      width: sizeStyle,
      transform: 'translateX(0)',
    },
    right: { 
      top: 0, 
      right: 0, 
      bottom: 0, 
      width: sizeStyle,
      transform: 'translateX(0)',
    },
    top: { 
      top: 0, 
      left: 0, 
      right: 0, 
      height: sizeStyle,
      transform: 'translateY(0)',
    },
    bottom: { 
      bottom: 0, 
      left: 0, 
      right: 0, 
      height: sizeStyle,
      transform: 'translateY(0)',
    },
  };
  
  return createPortal(
    <div 
      className="drawer-backdrop"
      onClick={onClose}
      role="presentation"
      style={{
        position: 'fixed',
        top: 0,
        left: 0,
        right: 0,
        bottom: 0,
        background: 'rgba(0, 0, 0, 0.5)',
        zIndex: 1000,
        animation: 'fadeIn 0.2s',
      }}
    >
      <div 
        ref={drawerRef}
        className={`drawer drawer-${placement}`}
        role="dialog"
        aria-modal="true"
        aria-labelledby={title ? titleId : undefined}
        onClick={(e) => e.stopPropagation()}
        style={{
          position: 'fixed',
          background: 'white',
          boxShadow: '0 0 16px rgba(0, 0, 0, 0.2)',
          padding: 24,
          overflow: 'auto',
          animation: `slideIn-${placement} 0.3s`,
          ...placementStyles[placement],
        }}
      >
        {title && (
          <header style={{ display: 'flex', justifyContent: 'space-between', marginBottom: 16 }}>
            <h2 id={titleId}>{title}</h2>
            <button onClick={onClose} aria-label="Close drawer">×</button>
          </header>
        )}
        <div className="drawer-body">{children}</div>
      </div>
    </div>,
    document.body
  );
}

// CSS (separate file)
/*
@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

@keyframes slideIn-left {
  from { transform: translateX(-100%); }
  to { transform: translateX(0); }
}

@keyframes slideIn-right {
  from { transform: translateX(100%); }
  to { transform: translateX(0); }
}

@keyframes slideIn-top {
  from { transform: translateY(-100%); }
  to { transform: translateY(0); }
}

@keyframes slideIn-bottom {
  from { transform: translateY(100%); }
  to { transform: translateY(0); }
}

// Accessibility — animation foydalanuvchi reduced motion ni xohlasa o'chirish (WCAG 2.3.3)
@media (prefers-reduced-motion: reduce) {
  .drawer-backdrop,
  .drawer {
    animation: none;
  }
}
*/

// Usage
function App() {
  const [isFilterOpen, setIsFilterOpen] = useState(false);
  
  return (
    <>
      <button onClick={() => setIsFilterOpen(true)}>Open Filters</button>
      
      <Drawer
        isOpen={isFilterOpen}
        onClose={() => setIsFilterOpen(false)}
        placement="right"
        size={350}
        title="Filters"
      >
        <div>
          <h3>Category</h3>
          <select>
            <option>All</option>
            <option>Electronics</option>
          </select>
          
          <h3>Price Range</h3>
          <input type="range" min="0" max="1000" />
          
          <button onClick={() => setIsFilterOpen(false)}>Apply</button>
        </div>
      </Drawer>
    </>
  );
}
```

**Tushuntirish:**
- Portal'da body'ga render — z-index, transform, overflow constraints'dan qutulish.
- Focus management: initial focus + Tab key trap + return focus.
- Body scroll lock — Drawer ochiq paytida arqa scroll bloklash.
- ARIA: `role="dialog"`, `aria-modal="true"`, `aria-labelledby`.
- CSS animations: backdrop fade + drawer slide.
- Placement-aware styles + animations (left/right/top/bottom).

**Production'da qo'shimcha:**
- `framer-motion` orqali physics-based animation.
- `react-focus-lock` library — battle-tested focus trap.
- iOS-safe scroll lock (`position: fixed` workaround).
- Resize handle — drag-to-resize drawer width.

</details>

---

## Xulosa

Portal — React'ning component'larni DOM tree boshqa joyiga render qiluvchi mexanizmi. Modal, Tooltip, Dropdown, Toast — production UI'larning fundamental primitive'lari shu pattern'ga asoslanadi. Asosiy fikrlar:

- **Portal Concept** — `createPortal(children, container)` API komponent JSX joyida yoziladi (logical placement), lekin DOM'da boshqa container ichida render qilinadi (visual placement). Body'ga render — overflow/z-index/transform parent constraints'dan qutulish. **R16 (2017)** introduced.
- **`createPortal` API** — `createPortal(children, container, key?)`. Container DOM Element yoki DocumentFragment. Key Reconciliation siblings uchun. Return — React Portal element (`$$typeof: REACT_PORTAL_TYPE`).
- **DOM Tree vs React Tree — fundamental farq** — Portal **DOM tree va React tree ajratadi**. DOM'da body'ga jump, React tree'da parent komponent ichida. Context inheritance, event bubbling, error boundaries, Reconciliation — **React tree bo'ylab** (DOM tree emas).
- **Event Bubbling React Tree Bo'ylab** — surprising advanced detail. Portal'd button click → React tree parent (App) bilan bubble (DOM tree body emas). R17+ event delegation root container'da, Fiber.return chain handler'larni dispatch qiladi. `stopPropagation` event handler'da bubbling block.
- **Real-World: Modal** — Portal + backdrop click + Escape key + focus trap + focus return + body scroll lock + ARIA `role="dialog" aria-modal="true"`. Production-grade implementation. iOS Safari `position: fixed` workaround scroll lock uchun. Compound Components Modal (Modal.Header/Body/Footer/Close).
- **Real-World: Tooltip** — Portal + hover/focus listeners + position calculation (`getBoundingClientRect`) + auto-flip viewport edge'da + delay (UX). `useLayoutEffect` sync DOM read. `@floating-ui/react` library production tavsiya.
- **Real-World: Dropdown / Popover** — Portal + click outside (`useOnClickOutside`) + Escape key + position + ARIA `aria-haspopup="true" aria-expanded`. Trigger ref + popup ref ikkalasi `contains` check Portal click outside uchun.
- **Real-World: Toast / Notification** — Context Provider (global state) + Portal (top-right corner) + queue management + auto-dismiss `setTimeout` + `framer-motion` animation. `toast.success/error/info/warning` API.
- **z-index va Stacking Contexts** — Portal stacking context muammoni hal qiladi. Parent `transform`/`opacity`/`filter` stacking context yaratadi → child `z-index: 9999` parent context tashqarisida ko'rinmaydi. Body'ga render — top-level stacking. CSS variables centralized z-index management. **`<dialog>` element + top layer** — modern alternative (cross-browser parity 2022 yildan: Firefox 98, Safari 15.4).
- **Focus Management va Focus Trap** — initial focus (birinchi focusable), focus trap (Tab key cycle), focus return (close paytida triggering element'ga). `useFocusTrap` + `useFocusReturn` hooks. HTML `inert` attribute — background elements'larni focus + events ignore (Chrome 102+ Firefox 112+ Safari 15.5+; React 19'da boolean prop proper render). `react-focus-lock` library production tavsiya.
- **SSR Considerations** — server'da `document.body` yo'q, `createPortal` xato beradi. Patterns: `mounted` state (`useEffect` post-mount), `typeof window` check, Next.js `dynamic({ ssr: false })`. Portal client-only render. Hydration mismatch yo'q (server Portal'siz, client mount paytida qo'shadi).
- **Portal Cleanup va Lifecycle** — Reconciler children'larni container'dan automatic remove qiladi unmount paytida. Custom container — `useEffect` cleanup'da manual `removeChild` shart (memory leak oldini olish). Shared container counter-based cleanup pattern.
- **Edge Cases** — `event.target` DOM tree (Portal'd element), `event.currentTarget` React tree handler, `ref` DOM joyini reflect qiladi, CSS `:hover` parent selector ishlamaydi, multiple Portals `key` Reconciliation, nested Portals OK (React tree nested, DOM tree flat).

Versiya evolyutsiyasi:
- **Pre-R16:** Portal mavjud emas, manual DOM manipulation workarounds.
- **R16 (2017):** `createPortal` API kiritildi.
- **R18 (2022):** Concurrent rendering bilan ishlaydi, Suspense Boundary va Error Boundary Portal ichidan oshib o'tadi.
- **R19 (2024):** `<title>`/`<meta>`/`<link>` document metadata Portal'siz `<head>`'ga hoist (cross-ref [`37-react-19-document-apis.md`](37-react-19-document-apis.md)).

Cross-references:

- [`02-rendering.md`](02-rendering.md) — Render+Commit Phases
- [`13-event-handling.md`](13-event-handling.md) — Event delegation R17+ root container
- [`17-uselayouteffect.md`](17-uselayouteffect.md) — `useLayoutEffect` Tooltip positioning, HTML `inert` attribute
- [`19-usecontext.md`](19-usecontext.md) — Context Provider through Portal
- [`22-concurrent-hooks.md`](22-concurrent-hooks.md) — `useId` SSR-safe IDs
- [`24-custom-hooks.md`](24-custom-hooks.md) — `useOnClickOutside`, `useEventListener`
- [`26-compound-components.md`](26-compound-components.md) — Modal/Dialog Compound Components
- [`27-error-boundaries.md`](27-error-boundaries.md) — Error Boundary Portal integration
- [`37-react-19-document-apis.md`](37-react-19-document-apis.md) — R19 Document Metadata

---

**Keyingi bo'lim:** [29-suspense-lazy.md](29-suspense-lazy.md) — Suspense va Lazy Loading: code splitting (`React.lazy` + `Suspense`), R19 `use()` + Suspense Promise handling, nested Suspense boundaries (granular loading states), loading states fallback (Skeleton/Spinner patterns), Suspense for data fetching R19 pattern, streaming SSR Suspense bilan progressive rendering, `SuspenseList` experimental status.
