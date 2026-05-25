# Bo'lim 13: Event Handling

> Event handling — foydalanuvchi action'lariga (click, input, submit, keyboard) komponent javob berishi. React'da `SyntheticEvent` cross-browser normalization, event delegation root container'ga (R17+), event pooling olib tashlanishi (R17), R19'dagi `<form action={fn}>` client-side actions, va TypeScript event types yoritiladi.

---

## Mundarija

- [Event Handler Asoslari](#event-handler-asoslari)
- [SyntheticEvent — Cross-Browser Normalization](#syntheticevent--cross-browser-normalization)
- [Event Object — `target`, `currentTarget`, `nativeEvent`](#event-object--target-currenttarget-nativeevent)
- [Event Delegation — R16 vs R17+](#event-delegation--r16-vs-r17)
- [Event Pooling — R17'da Olib Tashlandi](#event-pooling--r17da-olib-tashlandi)
- [Argument Passing va Naming Convention](#argument-passing-va-naming-convention)
- [Inline vs Separate Handler — Trade-off](#inline-vs-separate-handler--trade-off)
- [Propagation va Default Behavior](#propagation-va-default-behavior)
- [Common Events Catalog](#common-events-catalog)
- [R19 `<form action={fn}>` — Client-Side Actions](#r19-form-actionfn--client-side-actions)
- [TypeScript Event Types](#typescript-event-types)
- [TypeScript Event Narrowing va Generic Handlers](#typescript-event-narrowing-va-generic-handlers)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Event Handler Asoslari

### Nazariya

React'da event handler — JSX atribut sifatida camelCase nom bilan e'lon qilinadi (`onClick`, `onChange`, `onSubmit`). Qiymat — **function reference** bo'lishi shart.

```tsx
function Button() {
  const handleClick = () => {
    console.log('Clicked');
  };
  
  return <button onClick={handleClick}>Click</button>;
  //                       ↑ function reference (call qilinmagan)
}
```

**Eng katta xato — `onClick={handleClick()}`:**

```tsx
// ❌ Function chaqirilgan, return value (undefined) onClick'ga uzatilgan
<button onClick={handleClick()}>Click</button>
// React: handleClick darhol chaqiriladi (render paytida!)
// natija: undefined onClick'ga, click hech qanday ishlamaydi
// + render'da side effect (handleClick body bajarildi) — purity buzildi
```

**Function reference vs function call:**

```tsx
const handler = () => console.log('hello');

handler        // function reference
handler()      // function chaqiriladi, return value (undefined)

<button onClick={handler}>OK</button>     // ✅
<button onClick={handler()}>Bad</button>  // ❌
```

**Event handler signature:**

```tsx
function handleClick(event: React.MouseEvent<HTMLButtonElement>) {
  console.log(event.clientX, event.clientY);
}

<button onClick={handleClick}>Click</button>
```

React event object'ni argument sifatida uzatadi. TypeScript — `React.MouseEvent<HTMLButtonElement>` (cross-ref TS section).

**HTML vs React event handling:**

| Xususiyat | HTML | React |
|-----------|------|-------|
| Atribut nomi | `onclick` (lowercase) | `onClick` (camelCase) |
| Qiymat | String code | Function reference |
| `false` qaytaring → preventDefault | ✅ | ❌ (`e.preventDefault()`) |
| Multiple handlers | DOM API (addEventListener) | Single prop (override) |

```html
<!-- HTML -->
<button onclick="handleClick()">Click</button>

<!-- React JSX -->
<button onClick={handleClick}>Click</button>
```

**Inline arrow function:**

```tsx
<button onClick={() => console.log('Clicked')}>Click</button>

<button onClick={() => handleClick(item.id)}>Click</button>
// Argument'lar bilan handler chaqirish — wrapper kerak
```

Inline arrow — har render'da yangi function reference. Ko'pchilik holatda muammo emas, lekin `React.memo` child'larda re-render trigger qiladi (cross-ref [`33-optimization.md`](33-optimization.md)).

**Boolean condition'lar:**

```tsx
// ❌ Conditional null event
<button onClick={isDisabled ? null : handleClick}>Click</button>
// React: null OK, lekin disabled attribute afzal

// ✅ Disabled attribute
<button onClick={handleClick} disabled={isDisabled}>Click</button>

// ✅ Conditional handler logic
const handleClick = () => {
  if (isDisabled) return;
  // ... real logic
};
```

<details>
<summary><strong>Under the Hood</strong></summary>

JSX transform event handler:

```tsx
<button onClick={handleClick}>Click</button>

// Transform
_jsx('button', {
  onClick: handleClick,  // function reference
  children: 'Click',
});
```

`onClick` — props object'ning property'si. React'ning DOM Renderer (`react-dom`) bu prop'ni Commit Phase'da DOM'ga apply qiladi:

```ts
// react-dom internal (soddalashtirilgan)
function setProperty(node: Element, name: string, value: any) {
  if (name.startsWith('on')) {
    // Event handler
    const eventName = name.slice(2).toLowerCase();  // 'click'
    // Lekin React event'larni elementga to'g'ridan-to'g'ri qo'ymaydi
    // Root container'ga delegate qiladi (cross-ref Event Delegation section)
    addRootEventListener(eventName, node);
  } else {
    node.setAttribute(name, value);
  }
}
```

**Event handler reference identity:**

```tsx
function Component() {
  const handleClick = () => console.log('click');
  // Har render'da yangi function reference
  
  return <button onClick={handleClick}>Click</button>;
}
```

Har render'da `handleClick` yangi function — DOM event listener qaytadan attach qilinmaydi (chunki React root delegation), lekin Reconciler `React.memo`'lar bilan ishlamaydi:

```tsx
const Memo = React.memo(({ onClick }: { onClick: () => void }) => {
  return <button onClick={onClick}>Click</button>;
});

function Parent() {
  const handleClick = () => console.log('click');
  return <Memo onClick={handleClick} />;
  // Har Parent render'da handleClick yangi reference → Memo bailout ishlamaydi
}

// useCallback bilan stabilize:
function Parent() {
  const handleClick = useCallback(() => console.log('click'), []);
  return <Memo onClick={handleClick} />;
}
```

Cross-ref [`21-usememo-usecallback.md`](21-usememo-usecallback.md), [`33-optimization.md`](33-optimization.md).

**JSX transform name conversion:**

JSX uses camelCase, lekin DOM event'lari lowercase:

```
JSX: onClick   →  DOM event: 'click'
JSX: onChange  →  DOM event: 'change'
JSX: onMouseEnter → DOM event: 'mouseenter'
```

React internal'da bu mapping `eventTypes` table orqali (`react-dom` paket). Custom event'lar uchun `addEventListener` kerak (cross-ref [`38-web-components.md`](38-web-components.md)).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Eng oddiy event handler:

```tsx
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);
  
  const handleClick = () => {
    setCount((c) => c + 1);
  };
  
  return (
    <button onClick={handleClick}>
      Count: {count}
    </button>
  );
}
```

Argument'lar bilan handler:

```tsx
type Item = { id: number; name: string };

function ItemList({ items }: { items: Item[] }) {
  const handleSelect = (id: number) => {
    console.log('Selected:', id);
  };
  
  return (
    <ul>
      {items.map((item) => (
        <li key={item.id}>
          {item.name}
          <button onClick={() => handleSelect(item.id)}>Select</button>
          {/* ↑ Inline wrapper — ID'ni handler'ga uzatish uchun */}
        </li>
      ))}
    </ul>
  );
}
```

Multiple handlers — alohida funksiyalar:

```tsx
function FormButtons({ onSave, onCancel, onDelete }: FormButtonsProps) {
  const handleSave = (e: React.MouseEvent) => {
    e.preventDefault();
    onSave();
  };
  
  const handleCancel = () => {
    if (window.confirm('Discard changes?')) {
      onCancel();
    }
  };
  
  const handleDelete = () => {
    if (window.confirm('Are you sure?')) {
      onDelete();
    }
  };
  
  return (
    <Inline gap={8}>
      <button onClick={handleSave}>Save</button>
      <button onClick={handleCancel}>Cancel</button>
      <button onClick={handleDelete}>Delete</button>
    </Inline>
  );
}
```

Anti-pattern — function call:

```tsx
function HandlerCallAntiPattern() {
  const handleClick = () => console.log('hello');
  
  return (
    <>
      {/* ❌ handleClick darhol chaqiriladi (render paytida) */}
      <button onClick={handleClick()}>Anti 1</button>
      {/* "hello" console'da har render'da chiqadi */}
      
      {/* ❌ Same problem with arguments */}
      <button onClick={handleClickWithArg('x')}>Anti 2</button>
      
      {/* ✅ To'g'ri — wrapper bilan */}
      <button onClick={() => handleClickWithArg('x')}>OK</button>
    </>
  );
}
```

</details>

---

## SyntheticEvent — Cross-Browser Normalization

### Nazariya

React'da event object — `SyntheticEvent`. Bu — native DOM event'ni o'rab oladigan **wrapper**, cross-browser API farq'larini bartaraf qiladi.

```tsx
function Button() {
  const handleClick = (event: React.MouseEvent<HTMLButtonElement>) => {
    console.log(event.clientX);     // ← SyntheticEvent property
    console.log(event.target);       // ← SyntheticEvent property
    console.log(event.nativeEvent); // ← Native browser event
  };
  
  return <button onClick={handleClick}>Click</button>;
}
```

**Nima uchun SyntheticEvent kerak:**

1. **Cross-browser consistency** — IE, Safari, Chrome, Firefox API farq'larini bir xil qiladi
2. **W3C compliance** — modern API'ga moslash (eski browser'larda emulation)
3. **React-specific features** — `e.persist()` (R16), event pooling (eskirgan, R17)
4. **Type safety** — TypeScript `SyntheticEvent<E>` generic E (element type)

**SyntheticEvent ierarxiya:**

```
SyntheticEvent (base)
├─ MouseEvent        (click, mouseenter, mouseleave, ...)
├─ KeyboardEvent     (keydown, keyup, keypress)
├─ ChangeEvent       (change — inputs)
├─ FormEvent         (submit, reset)
├─ FocusEvent        (focus, blur)
├─ TouchEvent        (touchstart, touchmove, touchend)
├─ PointerEvent      (pointerdown, pointermove, pointerup)
├─ DragEvent         (drag, dragstart, dragend, drop)
├─ ClipboardEvent    (copy, cut, paste)
├─ AnimationEvent    (animationstart, animationend)
└─ TransitionEvent   (transitionend)
```

Har event tipi — alohida properties:
- `MouseEvent` → `clientX`, `clientY`, `button`
- `KeyboardEvent` → `key`, `code`, `shiftKey`, `ctrlKey`
- `ChangeEvent` → `target` (input element)

**SyntheticEvent vs Native Event:**

```tsx
function handleClick(event: React.MouseEvent<HTMLButtonElement>) {
  // SyntheticEvent properties — cross-browser normalized
  event.clientX;
  event.target;
  event.preventDefault();
  
  // Native event — browser API
  event.nativeEvent;  // MouseEvent (DOM)
  event.nativeEvent.composedPath();  // Event'ning method'i, SyntheticEvent'da yo'q
}
```

**Modern browser convergence:**

R16'da SyntheticEvent qiymati katta edi (IE9 va eski browser'lar). Modern browser'larda (Chrome, Edge, Firefox, Safari) DOM API ko'p joylarda standart. R17+ — SyntheticEvent yumshoq wrapper, kam normalization, ko'p properties native event'dan to'g'ridan-to'g'ri:

```tsx
// React 17+
event.target  // === event.nativeEvent.target (ko'p hollarda)
event.clientX // === event.nativeEvent.clientX
```

Lekin TypeScript types va `currentTarget` typing — React-specific, ishlatish davom etiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

SyntheticEvent definition (`@types/react`):

```ts
interface SyntheticEvent<E = Element, T = Event> extends BaseSyntheticEvent<T, EventTarget & E, EventTarget> {
}

interface BaseSyntheticEvent<E = Event, C = any, T = any> {
  nativeEvent: E;
  currentTarget: C;
  target: T;
  bubbles: boolean;
  cancelable: boolean;
  defaultPrevented: boolean;
  eventPhase: number;
  isTrusted: boolean;
  preventDefault(): void;
  stopPropagation(): void;
  isDefaultPrevented(): boolean;
  isPropagationStopped(): boolean;
  persist(): void;  // R17+ no-op (deprecated)
  timeStamp: number;
  type: string;
}
```

**Specific event types:**

```ts
interface MouseEvent<T = Element, E = NativeMouseEvent> extends UIEvent<T, E> {
  altKey: boolean;
  button: number;
  buttons: number;
  clientX: number;
  clientY: number;
  ctrlKey: boolean;
  metaKey: boolean;
  pageX: number;
  pageY: number;
  screenX: number;
  screenY: number;
  shiftKey: boolean;
  // ... va boshqalar
}

interface KeyboardEvent<T = Element> extends UIEvent<T, NativeKeyboardEvent> {
  altKey: boolean;
  charCode: number;   // deprecated (use 'key')
  ctrlKey: boolean;
  key: string;        // 'a', 'Enter', 'Escape', ... (character/named key)
  keyCode: number;    // deprecated (use 'key' or 'code')
  code: string;       // 'KeyA', 'Enter', 'Escape', ... (physical key, layout-independent)
  shiftKey: boolean;
  metaKey: boolean;
  repeat: boolean;
  // ...
}
```

**SyntheticEvent creation — internal:**

```ts
// react-dom internal (soddalashtirilgan)
function createSyntheticEvent<E extends Event>(
  reactName: string,
  reactEventType: string,
  targetInst: Fiber,
  nativeEvent: E,
  nativeTarget: EventTarget
): SyntheticEvent {
  const event: SyntheticEvent = {
    type: reactEventType,
    target: nativeTarget,
    currentTarget: nativeTarget,
    nativeEvent,
    timeStamp: nativeEvent.timeStamp,
    defaultPrevented: nativeEvent.defaultPrevented,
    isTrusted: nativeEvent.isTrusted,
    
    preventDefault() {
      this.defaultPrevented = true;
      nativeEvent.preventDefault();
    },
    
    stopPropagation() {
      nativeEvent.stopPropagation();
    },
    
    persist() {},  // No-op (R17+)
    
    // ... copy specific properties from nativeEvent
  };
  
  return event;
}
```

**`currentTarget` vs native event:**

```tsx
// Native event — currentTarget event handler joylashgan element
// React: currentTarget — virtual (Fiber'ga bog'langan)

// Bubble path:
// <div onClick={parentHandler}>     ← currentTarget=div
//   <button onClick={childHandler}>  ← currentTarget=button (handler attach)
//     Click
//   </button>
// </div>

// Click button:
// 1. childHandler(e) — e.currentTarget = button
// 2. parentHandler(e) — e.currentTarget = div (bubble paytida o'zgaradi)
// 3. e.target — har doim button (click manbai)
```

`currentTarget` — React event handler attach qilgan element. `target` — click haqiqatan ham yuz bergan element.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

MouseEvent — clientX/clientY:

```tsx
function MouseTracker() {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  
  const handleMove = (e: React.MouseEvent<HTMLDivElement>) => {
    setPos({ x: e.clientX, y: e.clientY });
  };
  
  return (
    <div
      onMouseMove={handleMove}
      style={{ width: 400, height: 200, border: '1px solid' }}
    >
      Position: ({pos.x}, {pos.y})
    </div>
  );
}
```

KeyboardEvent — `key` vs `code`:

```tsx
function KeyboardListener() {
  const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
    if (e.key === 'Enter') {
      console.log('Enter pressed (any layout)');
    }
    if (e.code === 'KeyW' && e.shiftKey) {
      console.log('Shift+W (physical position)');
    }
    if (e.key === 'Escape') {
      e.currentTarget.blur();
    }
  };
  
  return <input onKeyDown={handleKeyDown} placeholder="Type something" />;
}

// `e.key` — character qiymati (layout-dependent: 'a' AZERTY 'q' bo'ladi)
// `e.code` — physical key (layout-independent: 'KeyA' har doim same)
// `e.keyCode` — deprecated, ishlatmang
```

ChangeEvent — input value:

```tsx
function NameInput() {
  const [name, setName] = useState('');
  
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setName(e.target.value);
  };
  
  return (
    <input
      type="text"
      value={name}
      onChange={handleChange}
      placeholder="Name"
    />
  );
}
```

FormEvent — submit:

```tsx
function LoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();  // Browser'ning default submit'ni to'xtatadi
    console.log({ email, password });
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <Stack gap={8}>
        <input
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
        />
        <input
          type="password"
          value={password}
          onChange={(e) => setPassword(e.target.value)}
        />
        <button type="submit">Login</button>
      </Stack>
    </form>
  );
}
```

Native event API — `nativeEvent`:

```tsx
function PointerTracker() {
  const handlePointerMove = (e: React.PointerEvent<HTMLDivElement>) => {
    // SyntheticEvent properties — React MouseEvent/PointerEvent type'lariga kiritilgan
    console.log(e.clientX, e.clientY);
    console.log(e.movementX, e.movementY);  // SyntheticEvent'da mavjud (MouseEvent uzaytmasi)
    
    // Native-only API'lar — SyntheticEvent'da yo'q
    console.log(e.nativeEvent.composedPath());  // Event method
    console.log((e.nativeEvent as any).layerX);  // Non-standard, ba'zi browser'larda
  };
  
  return <div onPointerMove={handlePointerMove}>Track</div>;
}
```

</details>

---

## Event Object — `target`, `currentTarget`, `nativeEvent`

### Nazariya

Event object'da uchta muhim property mavjud:

1. **`event.target`** — event qaerda yuz berdi (click manbai)
2. **`event.currentTarget`** — event handler qaerda attach qilingan
3. **`event.nativeEvent`** — original native browser event

```tsx
function ParentChild() {
  const handleClick = (e: React.MouseEvent<HTMLDivElement>) => {
    console.log('target:', e.target);                // Click yuz bergan element
    console.log('currentTarget:', e.currentTarget);  // div (handler attach)
    console.log('nativeEvent:', e.nativeEvent);      // Native MouseEvent
  };
  
  return (
    <div onClick={handleClick}>
      <button>Click me</button>
      <button>Or me</button>
    </div>
  );
}
```

User button'ni bossa:
- `e.target` = `<button>` (click manbai)
- `e.currentTarget` = `<div>` (parent — handler joylashgan)
- `e.nativeEvent` = Native MouseEvent

**Event bubbling:**

DOM event bubble qiladi — child'da yuz bergan event parent'larga ko'tariladi. Har bubble bosqichida `currentTarget` o'zgaradi:

```tsx
function BubbleDemo() {
  return (
    <div onClick={(e) => console.log('div:', e.currentTarget.tagName)}>
      <section onClick={(e) => console.log('section:', e.currentTarget.tagName)}>
        <button onClick={(e) => console.log('button:', e.currentTarget.tagName)}>
          Click
        </button>
      </section>
    </div>
  );
  
  // Click button:
  // button: BUTTON
  // section: SECTION
  // div: DIV
  // (target — har doim BUTTON, currentTarget — bubble path bo'ylab)
}
```

**`stopPropagation` — bubble to'xtatish:**

```tsx
const handleChildClick = (e: React.MouseEvent) => {
  e.stopPropagation();  // Parent'larga bubble qilmaydi
  console.log('child handled');
};
```

**`preventDefault` — browser default behavior to'xtatish:**

```tsx
const handleSubmit = (e: React.FormEvent) => {
  e.preventDefault();  // Form submit (page reload) to'xtatiladi
  // Custom logic
};

const handleLink = (e: React.MouseEvent) => {
  e.preventDefault();  // Link navigation to'xtatiladi
  // Custom navigation
};
```

**`event.target` typing — narrow:**

`event.target` tipi `EventTarget` — bu generic interface, specific properties yo'q. Specific element kerak bo'lsa — narrow:

```tsx
function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
  // e.target — HTMLInputElement (chunki ChangeEvent generic specified)
  console.log(e.target.value);  // ✅ value mavjud
}

function handleClick(e: React.MouseEvent<HTMLDivElement>) {
  // e.target — EventTarget (chunki click div'da bo'lishi yoki child'da)
  // Narrow kerak:
  if (e.target instanceof HTMLButtonElement) {
    console.log(e.target.disabled);  // ✅ HTMLButtonElement properties
  }
}
```

**`currentTarget` typing — qattiqroq:**

```tsx
function handleClick(e: React.MouseEvent<HTMLButtonElement>) {
  // e.currentTarget — HTMLButtonElement (handler attach qilgan element)
  console.log(e.currentTarget.disabled);  // ✅ Type-safe
}
```

`currentTarget` tipi — handler attach qilingan element bilan kafolatli mos. `target` — narrow kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

DOM event bubble path:

```
Native event flow:
  1. Capture phase: window → document → ... → target ancestors
  2. Target: target element
  3. Bubble phase: target → ... → document → window
```

React (R17+) listener'ni root container'ga qo'yadi (cross-ref Event Delegation section). Bubble qilingan event'lar root'ga yetganda React handler'larini chaqiradi:

```ts
// React event flow
1. Native event (e.g. click) DOM'da yuz beradi
2. Bubble qiladi root container'ga
3. React listener (root'da attach qilingan) trigger
4. React traverse Fiber tree (target Fiber'dan root'gacha)
5. Har Fiber uchun, agar onClick handler bo'lsa — chaqiriladi
6. SyntheticEvent har handler chaqiruvida currentTarget yangilanadi
```

**`currentTarget` mutation per handler:**

```tsx
<div onClick={divHandler}>
  <button onClick={buttonHandler}>Click</button>
</div>

// Click button:
// 1. buttonHandler(e) — e.currentTarget mutation: <button>
// 2. divHandler(e) — e.currentTarget mutation: <div>
```

**Async kontekstda `currentTarget` muammosi:**

```tsx
const handleClick = (e: React.MouseEvent) => {
  console.log('Sync:', e.currentTarget.tagName);  // BUTTON
  
  setTimeout(() => {
    console.log('Async:', e.currentTarget.tagName);
    // ⚠️ R16'da: e.currentTarget = null (event pooled)
    // R17+: e.currentTarget hali ham reference, lekin DOM'dan chiqarilgan bo'lishi mumkin
  }, 0);
};
```

R17+ event pooling olib tashlandi (keyingi section). Lekin `currentTarget` mutation handler chaqiruvi davomida — async kontekstda eski qiymat saqlanmaydi:

```tsx
const handleClick = (e: React.MouseEvent) => {
  const target = e.currentTarget;  // Snapshot
  setTimeout(() => {
    console.log(target.tagName);  // ✅ Snapshot saqlanadi
  }, 0);
};
```

**Native event reference:**

```tsx
const handleKeyDown = (e: React.KeyboardEvent) => {
  e.nativeEvent;  // Native KeyboardEvent
  
  e.nativeEvent.constructor === KeyboardEvent;  // true
  e.nativeEvent === e.nativeEvent;  // true (har doim bir reference)
};
```

`nativeEvent` — original DOM event reference. SyntheticEvent property'lari uning ichidan olinadi (R17+'da deyarli to'g'ridan-to'g'ri reference).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

target vs currentTarget — delegated handler:

```tsx
function Toolbar({ onAction }: { onAction: (action: string) => void }) {
  // Single handler delegated for ko'p button
  const handleClick = (e: React.MouseEvent<HTMLDivElement>) => {
    if (e.target instanceof HTMLButtonElement) {
      const action = e.target.dataset.action;
      if (action) onAction(action);
    }
  };
  
  return (
    <div onClick={handleClick}>
      <button data-action="save">Save</button>
      <button data-action="cancel">Cancel</button>
      <button data-action="delete">Delete</button>
    </div>
  );
  // ✅ Bitta handler, har button'ga alohida onClick kerak emas
  // e.target — bosilgan button, e.currentTarget — div (handler attach)
}
```

stopPropagation — modal close click handling:

```tsx
type ModalProps = {
  children: React.ReactNode;
  onClose: () => void;
};

function Modal({ children, onClose }: ModalProps) {
  return (
    <div className="overlay" onClick={onClose}>
      {/* Click overlay → onClose */}
      <div className="modal" onClick={(e) => e.stopPropagation()}>
        {/* Click modal content → bubble overlay'ga to'xtaydi */}
        {children}
      </div>
    </div>
  );
}
```

preventDefault — link navigation:

```tsx
function CustomLink({ href, children }: { href: string; children: React.ReactNode }) {
  const handleClick = (e: React.MouseEvent<HTMLAnchorElement>) => {
    e.preventDefault();
    // Custom navigation (e.g. router.push)
    history.pushState({}, '', href);
    window.dispatchEvent(new PopStateEvent('popstate'));
  };
  
  return (
    <a href={href} onClick={handleClick}>
      {children}
    </a>
  );
}
```

Native event — modern API:

```tsx
function PointerArea() {
  const handlePointerMove = (e: React.PointerEvent<HTMLDivElement>) => {
    // Native API — pointer event ko'rsatkichlari
    const native = e.nativeEvent;
    console.log({
      pointerType: native.pointerType,  // 'mouse', 'touch', 'pen'
      pressure: native.pressure,         // 0-1 (touch/pen)
      tiltX: native.tiltX,              // pen tilt
      width: native.width,               // contact area
      height: native.height,
    });
  };
  
  return <div onPointerMove={handlePointerMove}>Track pointer</div>;
}
```

Async kontekst — currentTarget snapshot:

```tsx
function AsyncHandler() {
  const handleClick = async (e: React.MouseEvent<HTMLButtonElement>) => {
    const button = e.currentTarget;  // Snapshot — async paytida saqlanadi
    
    button.disabled = true;
    
    try {
      await fetch('/api/save');
      button.textContent = 'Saved!';
    } finally {
      button.disabled = false;
    }
    
    // ❌ Snapshot'siz: e.currentTarget async paytida null/stale
    // setTimeout(() => e.currentTarget.disabled = false, 1000);  // ⚠️
  };
  
  return <button onClick={handleClick}>Save</button>;
}
```

</details>

---

## Event Delegation — R16 vs R17+

### Nazariya

**Event delegation** — child element'lardagi event'larni parent'dagi listener orqali ushlash. React har event handler uchun **alohida DOM listener qo'ymaydi**, balki **single listener** higher-level node'ga qo'yadi.

> **🕐 Versiya evolyutsiyasi (Event Delegation):**
> - **R16 va undan oldin:** React `document` node'ga delegate qilardi. Bu — single React app uchun OK, lekin **multiple React apps bir page'da** bo'lsa, document'ga ikki listener registratsiya qilinardi va `stopPropagation` semantikasi noaniq edi.
> - **R17+:** **Root container** (`ReactDOM.render(<App/>, container)` yoki R18+ `createRoot(container)`'da `container`) ga delegate qiladi. Har React app — o'z root container'iga listener qo'yadi, mustaqil ishlaydi.
> - **Sabab:** **Microfrontends** va gradual upgrade support — bir page'da bir nechta React versiya yoki React + boshqa framework integration. R17+ — har app o'z scope'ida event handle qiladi va eski versiya bilan birga ishlashda collision kamayadi.

```tsx
// User code
<button onClick={handleClick}>Click</button>

// R16 internal:
// document.addEventListener('click', reactDispatcher);
// — barcha click'lar document'da ushlangan

// R17+ internal:
// root.addEventListener('click', reactDispatcher);
// — faqat root ichidagi click'lar ushlangan
```

**Microfrontends scenario:**

```tsx
// App 1 — main app
createRoot(document.getElementById('app1')!).render(<MainApp />);

// App 2 — embedded widget
createRoot(document.getElementById('app2')!).render(<Widget />);

// R16: document'ga ikki listener — collision, event order noaniq
// R17+: har root'ga alohida listener — mustaqil ishlaydi
```

**Native listener bilan integration:**

```tsx
// 3rd-party library native listener qo'yadi:
document.addEventListener('click', (e) => {
  console.log('Native handler');
});

function ReactComponent() {
  return <button onClick={() => console.log('React handler')}>Click</button>;
}
```

**R16 (eski):**
- Native listener `document`'da
- React listener ham `document`'da
- Order noaniq — ikkala handler bir vaqtda chaqiriladi

**R17+ (yangi):**
- Native listener `document`'da
- React listener root container'da
- Click button → React handler birinchi (root → ... → button bubble), keyin native handler (document'gacha bubble)

**`stopPropagation` semantikasi farqli:**

```tsx
function ReactComponent() {
  const handleClick = (e: React.MouseEvent) => {
    e.stopPropagation();
  };
  return <button onClick={handleClick}>Click</button>;
}
```

Asosiy farq — listener qaysi DOM node'da:

**R16:** React'ning o'zi `document`'ga listener qo'ygan. Bir click event document'gacha bubble qiladi, React Fiber tree traverse qiladi, `e.stopPropagation()` Fiber chain'da bubble'ni to'xtatadi va `nativeEvent.stopPropagation()` ham chaqiriladi (bu document'dan keyingi window'gacha bubble'ni to'xtatadi). Lekin **document'da React'dan oldin registered native listener'lar** — `document.addEventListener` order bo'yicha React'dan oldin chaqiriladi → ularni `e.stopPropagation()` to'xtata olmaydi.

**R17+:** React listener root container'da. Bubble path: `target → ... → root → document → window`. React handler root'ga yetganda chaqiriladi, `e.stopPropagation()` synthetic chain va native event ikkalasini to'xtatadi → document/window'gacha bubble qilmaydi. Lekin **agar bir element React render qilgan tree ichida bo'lmasa** (e.g., portal'siz JSX'siz DOM manipulation), React event delegation ushlamaydi.

Bu — backward incompatible o'zgarish. Migration paytida e'tibor:

```tsx
// R17+ — native listener'gacha bubble to'xtatish
const handleClick = (e: React.MouseEvent) => {
  e.stopPropagation();              // React layer
  e.nativeEvent.stopPropagation();  // Native layer ham
};
```

**Event delegation foyda:**

1. **Memory** — har element uchun alohida listener emas, bitta delegate
2. **Dynamic DOM** — yangi element qo'shilganda listener attach qilish kerak emas (delegate bubble path orqali ushlaydi)
3. **Performance** — listener attachment overhead kamayadi

<details>
<summary><strong>Under the Hood</strong></summary>

React event listener registration (R17+):

```ts
// react-dom internal (soddalashtirilgan)
function listenToAllSupportedEvents(rootContainerElement: Element) {
  for (const eventName of allNativeEvents) {
    listenToNativeEvent(eventName, false, rootContainerElement);
    listenToNativeEvent(eventName, true, rootContainerElement);  // capture
  }
}

function listenToNativeEvent(
  domEventName: string,
  isCapturePhase: boolean,
  target: Element
) {
  const listener = createEventListenerWrapper(target, domEventName, ...);
  if (isCapturePhase) {
    target.addEventListener(domEventName, listener, true);
  } else {
    target.addEventListener(domEventName, listener, false);
  }
}
```

Har root container uchun — barcha event tip'lari uchun listener qo'yiladi (capture + bubble fazalari). Listener — generic dispatcher.

**Dispatcher ishlash:**

```ts
function dispatchEvent(domEventName: string, eventSystemFlags: number, nativeEvent: Event) {
  // 1. Target Fiber'ni topish (DOM node → Fiber)
  const nativeEventTarget = nativeEvent.target as Element;
  const targetFiber = getClosestInstanceFromNode(nativeEventTarget);
  
  // 2. SyntheticEvent yaratish
  const syntheticEvent = createSyntheticEvent(...);
  
  // 3. Fiber tree bo'ylab yuqoriga walk — har handler chaqirish
  let fiber = targetFiber;
  while (fiber !== null) {
    if (hasEventListener(fiber, domEventName)) {
      const listener = getEventListener(fiber, domEventName);
      syntheticEvent.currentTarget = fiber.stateNode;
      listener(syntheticEvent);
      
      if (syntheticEvent.isPropagationStopped()) break;
    }
    fiber = fiber.return;  // parent
  }
}
```

**Event delegation va Fiber tree:**

React bubble path'ni **Fiber tree** orqali simulyatsiya qiladi (DOM bubble emas):

```
DOM:
  document
    div#root
      App
        Card
          Button (click)

Fiber tree walk (target → root):
  Button.onClick
  Card.onClick (yo'q — skip)
  App.onClick (yo'q — skip)
  div#root (root)

React: Button.onClick → Card.onClick → ... (oldindan registered handler'lar)
```

**Delegation cheklov — non-bubbling event'lar:**

Ba'zi event'lar bubble qilmaydi: `mouseenter`, `mouseleave`, `focus`, `blur`. React bularni manually emulyatsiya qiladi (mouseover/mouseout dan derive):

```ts
// React internal
'onMouseEnter' → derived from 'mouseover' (bubble qiladi) + ancestor check
'onMouseLeave' → derived from 'mouseout' + ancestor check
```

`onFocus`, `onBlur` — `focusin`/`focusout` (bubble qiladi) ishlatiladi.

**iframe va Shadow DOM:**

Event delegation iframe va Shadow DOM bilan murakkab:
- iframe — ayrim event boundary
- Shadow DOM — event retarget (open vs closed mode)

Bu cheklov'lar — kursdan tashqari (advanced topic).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Microfrontend scenario:

```tsx
// app1.tsx
import { createRoot } from 'react-dom/client';
import MainApp from './MainApp';

createRoot(document.getElementById('app1')!).render(<MainApp />);

// app2.tsx
import { createRoot } from 'react-dom/client';
import Widget from './Widget';

createRoot(document.getElementById('app2')!).render(<Widget />);

// HTML:
// <div id="app1"></div>
// <div id="app2"></div>
// R17+: har root o'z scope'ida event handle qiladi
```

Native + React listener integration:

```tsx
import { useEffect } from 'react';

function GlobalShortcuts() {
  useEffect(() => {
    const handleGlobalKey = (e: KeyboardEvent) => {
      if (e.key === '/' && e.ctrlKey) {
        e.preventDefault();
        // Open search
      }
    };
    
    document.addEventListener('keydown', handleGlobalKey);
    return () => document.removeEventListener('keydown', handleGlobalKey);
  }, []);
  
  return (
    <input
      onKeyDown={(e) => {
        // React handler — root container scope
        if (e.key === '/') {
          e.stopPropagation();
          e.nativeEvent.stopPropagation();
          // ↑ R17+: React layer + native layer
        }
      }}
    />
  );
}
```

Delegated event handling — list:

```tsx
type Item = { id: number; name: string };

function ItemList({ items, onSelect }: { items: Item[]; onSelect: (id: number) => void }) {
  // Har item uchun alohida onClick yozish o'rniga delegate
  const handleListClick = (e: React.MouseEvent<HTMLUListElement>) => {
    if (e.target instanceof HTMLLIElement) {
      const id = Number(e.target.dataset.id);
      if (id) onSelect(id);
    }
  };
  
  return (
    <ul onClick={handleListClick}>
      {items.map((item) => (
        <li key={item.id} data-id={item.id}>
          {item.name}
        </li>
      ))}
    </ul>
  );
}
```

R16 vs R17+ stopPropagation — migration:

```tsx
// Document'da global listener
useEffect(() => {
  const handleDocumentClick = () => {
    console.log('Document handler');
  };
  document.addEventListener('click', handleDocumentClick);
  return () => document.removeEventListener('click', handleDocumentClick);
}, []);

const handleButtonClick = (e: React.MouseEvent) => {
  e.stopPropagation();
  // SyntheticEvent.stopPropagation() ichida nativeEvent.stopPropagation() ham chaqiriladi.
  //
  // R16: React listener document'da edi → React Fiber chain'ida event'ni handle qilgandan
  //   keyin document'da React listener'idan keyingi handler'lar ishlamaydi (lekin oldin
  //   registered bo'lganlar — registration order bo'yicha — ishlaydi).
  // R17+: React listener root container'da. Bubble path: target → root → document.
  //   React handler root'da chaqiriladi va stopPropagation native event'ni to'xtatadi —
  //   document handler ko'pchilik holatda chaqirilmaydi. Lekin React'dan tashqaridagi
  //   capture phase listener'lar yoki document'da React rendering tree'sidan tashqari
  //   yuz bergan event'lar — boshqa semantikaga ega.
  //
  // Eng aniq nazoratni xohlasangiz — `stopImmediatePropagation` (native, bir node'dagi
  // boshqa listener'larni ham to'xtatadi):
  //   e.nativeEvent.stopImmediatePropagation();
};
```

</details>

---

## Event Pooling — R17'da Olib Tashlandi

### Nazariya

**Event pooling** — R16 va undan oldingi versiyalardagi mexanizm: SyntheticEvent object'lar **pool**'da saqlanardi va re-use qilinardi. Memory optimization (har event uchun yangi object yaratmaslik).

**Muammo:** Pooled object'lar event handler tugaganidan keyin **null** qilinardi. Async kontekstda — `e.target`, `e.currentTarget` undefined.

```tsx
// R16 anti-pattern
function PooledEventAsyncDemo() {
  const handleClick = (e: React.MouseEvent) => {
    setTimeout(() => {
      console.log(e.target);  // ⚠️ R16: null (event pooled)
    }, 0);
  };
  return <button onClick={handleClick}>Click</button>;
}

// R16 yechim — e.persist():
function R16PersistDemo() {
  const handleClick = (e: React.MouseEvent) => {
    e.persist();  // Event'ni pool'dan chiqarish
    setTimeout(() => {
      console.log(e.target);  // ✅ Mavjud
    }, 0);
  };
  return <button onClick={handleClick}>Click</button>;
}
```

> **🕐 Versiya evolyutsiyasi (Event Pooling):**
> - **R16 va undan oldin:** SyntheticEvent pooled, async kontekstda `e.persist()` zaruriy edi.
> - **R17+:** Pooling olib tashlandi. Har event uchun yangi object. Async kontekstda `e.target` mavjud.
> - **`e.persist()`** R17+'da hali ham mavjud (backward compatibility), lekin **no-op** (hech narsa qilmaydi). Deprecated.
> - **Sabab:** Modern engine'lar GC tezroq, pooling foyda kichik. Code complexity (persist API) — kerak emas. Async kontekstdagi xatolar — silent bug'lar manbai edi.

**Migration:**

```tsx
// R16 kod:
const handleClick = (e: React.MouseEvent) => {
  e.persist();  // R17+: no-op
  setTimeout(() => console.log(e.target), 0);
};

// R17+ — e.persist() shart emas:
const handleClick = (e: React.MouseEvent) => {
  setTimeout(() => console.log(e.target), 0);  // ✅ Mavjud
};
```

**Async kontekstda saqlash — modern pattern:**

R17+'da snapshot pattern hali ham yaxshi praktika (kod aniqlik beradi):

```tsx
const handleClick = async (e: React.MouseEvent<HTMLButtonElement>) => {
  // Snapshot — async paytida ham aniq kuzatish
  const button = e.currentTarget;
  const buttonText = button.textContent;
  
  await delay(1000);
  
  console.log(`Clicked: ${buttonText}`);
  // Snapshot — currentTarget mutation'siz, button reference saqlanadi
};
```

**Interview savoli sifatida — legacy:**

Event pooling — interview'larda hali ham uchraydi (legacy knowledge). Modern React'da pooling yo'q, lekin:

- "Event pooling nima?" — R16'gacha SyntheticEvent re-use mexanizmi
- "Nima uchun olib tashlandi?" — Modern engine'lar, code complexity, async bug
- "`e.persist()` qachon kerak?" — R16'da; R17+'da no-op (deprecated)

<details>
<summary><strong>Under the Hood</strong></summary>

R16 event pooling internal:

```ts
// R16 react-dom internal
const eventPool: SyntheticEvent[] = [];

function getPooledEvent(...) {
  if (eventPool.length > 0) {
    return eventPool.pop()!;
  }
  return new SyntheticEvent();
}

function releaseEvent(event: SyntheticEvent) {
  // Reset all properties to null
  event.dispatchConfig = null;
  event.target = null;
  event.currentTarget = null;
  event.nativeEvent = null;
  // ...
  eventPool.push(event);
}

function dispatchEvent(...) {
  const event = getPooledEvent(...);
  // Handler chaqiriladi (sync)
  invokeHandler(event);
  // Handler tugagandan so'ng — release
  releaseEvent(event);
  // Async kontekstda eski reference event — hammasi null
}
```

`event.persist()` — pool'dan chiqarish:

```ts
SyntheticEvent.prototype.persist = function() {
  this._isPersistent = true;
  // releaseEvent skip qiladi agar _isPersistent
};
```

R17+ pooling olib tashlangan:

```ts
// R17+ — no pooling
function dispatchEvent(...) {
  const event = new SyntheticEvent(...);  // har safar yangi
  invokeHandler(event);
  // Release yo'q — GC handle qiladi
}

SyntheticEvent.prototype.persist = function() {
  // No-op
};
```

**Performance implications:**

R16 pooling — har event uchun memory allocation kamaytirardi. Modern JS engine'lari short-lived object'lar uchun generational GC ishlatadi (young generation / nursery), shu sababli qisqa muddatli object'lar uchun allocation cost minimal. Pooling overhead (release logic, pool management) — modern engine'larda real foyda bermaydi. R17+ — sodda kod, GC handle qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

R17+ — async kontekstda e.target ishlaydi:

```tsx
function FetchOnClick() {
  const handleClick = async (e: React.MouseEvent<HTMLButtonElement>) => {
    const button = e.currentTarget;
    button.disabled = true;
    
    try {
      const response = await fetch('/api/data');
      const data = await response.json();
      
      // Async kontekst — e.target hali ham accessible
      console.log('Source button:', e.target);
      console.log('Handler attached to:', e.currentTarget);
    } finally {
      button.disabled = false;  // Snapshot reference
    }
  };
  
  return <button onClick={handleClick}>Fetch</button>;
}
```

R16 legacy code — migration:

```tsx
// R16 kod
function R16Component() {
  const handleClick = (e: React.MouseEvent) => {
    e.persist();  // R16'da zaruriy, R17+'da no-op
    
    setTimeout(() => {
      console.log(e.target);
    }, 1000);
  };
  
  return <button onClick={handleClick}>Click</button>;
}

// R17+ migration
function R17Component() {
  const handleClick = (e: React.MouseEvent) => {
    // e.persist() olib tashlandi (no-op, lekin code'ni tozalash)
    
    setTimeout(() => {
      console.log(e.target);  // ✅ Mavjud
    }, 1000);
  };
  
  return <button onClick={handleClick}>Click</button>;
}
```

Snapshot pattern — modern best practice:

```tsx
function ProgressTracker() {
  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    
    // Snapshot — clarity va explicit
    const form = e.currentTarget;
    const formData = new FormData(form);
    
    try {
      await fetch('/api/submit', {
        method: 'POST',
        body: formData,
      });
      
      form.reset();  // Snapshot ishlaydi (chunki form mavjud DOM'da)
    } catch (err) {
      console.error('Submit failed:', err);
    }
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input name="email" type="email" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

</details>

---

## Argument Passing va Naming Convention

### Nazariya

Event handler'larga qo'shimcha argument uzatish — wrapper function bilan:

```tsx
function handleSelect(id: number) {
  console.log('Selected:', id);
}

<button onClick={() => handleSelect(123)}>Select</button>
//                ↑ Inline arrow wrapper — id'ni handler'ga uzatish
```

**Inline arrow vs `bind`:**

```tsx
// Inline arrow (modern)
<button onClick={() => handleSelect(item.id)}>Select</button>

// Function.prototype.bind (eski)
<button onClick={handleSelect.bind(null, item.id)}>Select</button>
```

Ikkalasi ham har render'da yangi function reference yaratadi. Modern stil — arrow function (oddiyroq).

**Curried handler:**

```tsx
function makeSelectHandler(id: number) {
  return () => {
    console.log('Selected:', id);
  };
}

<button onClick={makeSelectHandler(item.id)}>Select</button>
// Bir xil natija — wrapper function qaytaradi
```

**Naming convention:**

React community'da standart pattern:

| Type | Pattern | Misol |
|------|---------|-------|
| **Prop** | `on<Action>` | `onClick`, `onSubmit`, `onSelect` |
| **Handler function** | `handle<Action>` | `handleClick`, `handleSubmit`, `handleSelect` |
| **Custom event prop** | `on<DomainAction>` | `onUserSelect`, `onItemRemove`, `onPageChange` |

```tsx
type Props = {
  onUserSelect: (userId: number) => void;
  // ↑ Prop: onAction (consumer perspective)
};

function UserList({ onUserSelect }: Props) {
  const handleSelect = (e: React.MouseEvent<HTMLLIElement>) => {
    // ↑ Internal handler: handleAction
    const userId = Number(e.currentTarget.dataset.userId);
    onUserSelect(userId);
  };
  
  return (
    <ul>
      {users.map((user) => (
        <li
          key={user.id}
          data-user-id={user.id}
          onClick={handleSelect}
        >
          {user.name}
        </li>
      ))}
    </ul>
  );
}
```

**Why this convention:**

1. **Consumer clarity** — `onUserSelect` — consumer perspective ("nima yuz beradi")
2. **Internal logic** — `handleSelect` — implementation perspective ("nima qilamiz")
3. **Convention community-wide** — boshqa loyihalar va kutubxonalar bilan moslashish

**Namespaced events — domain-specific:**

```tsx
type FormEvents = {
  onValueChange: (value: string) => void;
  onValidationError: (error: string) => void;
  onSubmitStart: () => void;
  onSubmitSuccess: (response: ApiResponse) => void;
  onSubmitError: (error: Error) => void;
};
```

Domain'ga moslashtirilgan event nom — UI mexanizm emas, business logic.

<details>
<summary><strong>Under the Hood</strong></summary>

**Inline arrow vs stable handler — performance:**

```tsx
function Parent() {
  return (
    <List
      items={items}
      onItemClick={(id) => console.log(id)}  // har render yangi reference
    />
  );
}

const Memo = React.memo(List);

// Memo bailout ishlamaydi — onItemClick har render yangi
// useCallback bilan stabilize:
function Parent() {
  const handleItemClick = useCallback((id: number) => console.log(id), []);
  return <Memo items={items} onItemClick={handleItemClick} />;
}
```

`React.memo` + `useCallback` — child re-render kamaytirish (cross-ref [`33-optimization.md`](33-optimization.md)). Lekin overhead `useCallback` bo'lgan ham bor (har render dependency check). Komponent size va render tezligiga qarab tanlash.

**Bind vs arrow performance:**

```ts
// bind — yangi function ham, har render yangi
const bound = handler.bind(null, arg);

// arrow — yangi function ham, har render yangi
const arrow = () => handler(arg);
```

V8 engine ikkala holatda yangi function object yaratadi. Performance farqi sezilmas. Arrow — readability afzal.

**Curried handler factory:**

```tsx
function makeHandler(domain: string) {
  return (id: number) => (e: React.MouseEvent) => {
    console.log(domain, id, e.target);
  };
}

const handleUserClick = makeHandler('user');

<button onClick={handleUserClick(123)}>Click</button>
```

Curried function — partial application. JS closure ko'p marta nested — performance overhead minimal, lekin readability murakkab. Ko'pchilik holatda — oddiy arrow wrapper afzal.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Argument passing patterns:

```tsx
type Item = { id: number; name: string };

function ItemList({ items, onSelect }: { items: Item[]; onSelect: (id: number) => void }) {
  return (
    <ul>
      {items.map((item) => (
        <li key={item.id}>
          <span>{item.name}</span>
          
          {/* Inline arrow */}
          <button onClick={() => onSelect(item.id)}>Select 1</button>
          
          {/* Data attribute + delegated */}
          <button
            data-id={item.id}
            onClick={(e) => {
              const id = Number(e.currentTarget.dataset.id);
              onSelect(id);
            }}
          >
            Select 2
          </button>
          
          {/* Curried handler */}
          <button onClick={makeSelectHandler(item.id, onSelect)}>Select 3</button>
        </li>
      ))}
    </ul>
  );
}

function makeSelectHandler(id: number, onSelect: (id: number) => void) {
  return () => onSelect(id);
}
```

Naming convention demo:

```tsx
type ProductCardProps = {
  product: Product;
  onProductSelect: (id: number) => void;     // ✅ Prop: on<DomainAction>
  onProductFavorite: (id: number) => void;
  onProductShare: (id: number) => void;
};

function ProductCard({ product, onProductSelect, onProductFavorite, onProductShare }: ProductCardProps) {
  // ✅ Internal handler: handle<Action>
  const handleSelectClick = () => {
    onProductSelect(product.id);
  };
  
  const handleFavoriteClick = (e: React.MouseEvent) => {
    e.stopPropagation();
    onProductFavorite(product.id);
  };
  
  const handleShareClick = async (e: React.MouseEvent) => {
    e.stopPropagation();
    onProductShare(product.id);
  };
  
  return (
    <div onClick={handleSelectClick}>
      <h3>{product.name}</h3>
      <Inline gap={4}>
        <button onClick={handleFavoriteClick}>♥ Favorite</button>
        <button onClick={handleShareClick}>↗ Share</button>
      </Inline>
    </div>
  );
}
```

useCallback for memo bailout:

```tsx
import { useCallback, memo } from 'react';

const ItemRow = memo(function ItemRow({ item, onClick }: { item: Item; onClick: (id: number) => void }) {
  return (
    <li>
      {item.name}
      <button onClick={() => onClick(item.id)}>Select</button>
    </li>
  );
});

function List({ items }: { items: Item[] }) {
  // ❌ Har render yangi function — memo bailout ishlamaydi
  // const handleClick = (id: number) => console.log(id);
  
  // ✅ useCallback — stable reference
  const handleClick = useCallback((id: number) => console.log(id), []);
  
  return (
    <ul>
      {items.map((item) => (
        <ItemRow key={item.id} item={item} onClick={handleClick} />
      ))}
    </ul>
  );
}
```

</details>

---

## Inline vs Separate Handler — Trade-off

### Nazariya

**Inline handler** — JSX ichida arrow function yoki anonymous function:

```tsx
<button onClick={() => setCount(c => c + 1)}>+</button>
```

**Separate handler** — function alohida e'lon qilingan:

```tsx
const handleClick = () => setCount(c => c + 1);
return <button onClick={handleClick}>+</button>;
```

**Trade-off:**

| Aspect | Inline | Separate |
|--------|--------|----------|
| Readability | Sodda case'da yaxshi | Murakkab logic uchun yaxshi |
| Reusability | Yo'q | Bir necha joyda ishlatish mumkin |
| Reference identity | Har render yangi | `useCallback` bilan stabilize |
| Test | Inline test qiyin | Alohida unit test |
| Performance | Bir xil (memo bilan farq) | useCallback overhead |

**Inline OK qachon:**

1. **Sodda logic** — `() => setOpen(true)`, `() => onSelect(item.id)`
2. **Bir martalik chaqiruv** — qaytadan ishlatilmaydi
3. **`React.memo` ishlatilmagan child** — performance impact yo'q

**Separate handler afzal qachon:**

1. **Murakkab logic** — bir nechta operation, validation, error handling
2. **Reusability** — bir handler turli joylarda
3. **Memoization needed** — `useCallback` bilan child re-render kamaytirish
4. **Testing** — handler'ni alohida test qilish

```tsx
// ❌ Inline murakkab logic — readability yomon
<form onSubmit={async (e) => {
  e.preventDefault();
  setSubmitting(true);
  try {
    const formData = new FormData(e.currentTarget);
    const response = await fetch('/api', { method: 'POST', body: formData });
    if (!response.ok) throw new Error('Submit failed');
    onSuccess(await response.json());
  } catch (err) {
    setError(err as Error);
  } finally {
    setSubmitting(false);
  }
}}>...</form>

// ✅ Separate handler
const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault();
  setSubmitting(true);
  try {
    const formData = new FormData(e.currentTarget);
    const response = await fetch('/api', { method: 'POST', body: formData });
    if (!response.ok) throw new Error('Submit failed');
    onSuccess(await response.json());
  } catch (err) {
    setError(err as Error);
  } finally {
    setSubmitting(false);
  }
};

return <form onSubmit={handleSubmit}>...</form>;
```

**Performance — myth vs reality:**

"Inline handler — performance yomon" — ko'pincha o'rta-tashqari claim. Real impact:

1. **DOM listener attach/detach yo'q** — React delegation orqali (root container'da single listener)
2. **Function creation cost** — minimal (V8 inline allocation, GC tezda yiqiladi)
3. **Object identity** — `React.memo` ishlatilmasa, ta'sir yo'q

Real performance impact — `React.memo` + child component'lar ko'p bo'lganda. Bu aniq holatlarda `useCallback` orqali optimize qilinadi.

**Profile birinchi, optimize keyin:**

```tsx
// Ko'pchilik holatda — inline OK
<button onClick={() => setOpen(true)}>Open</button>

// React DevTools Profiler bilan bottleneck topilsa — alohida + useCallback
```

<details>
<summary><strong>Under the Hood</strong></summary>

V8 (Chrome) inline arrow function — Hidden Class:

```ts
// JSX render
<button onClick={() => setOpen(true)}>Open</button>

// Compiled
const onClickHandler = () => setOpen(true);
React.createElement('button', { onClick: onClickHandler }, 'Open');

// Har render: yangi function object
// Hidden class: same shape (same closure variables)
// V8 optimize qiladi — function instantiation modern engine'larda tez
```

`useCallback` overhead:

```ts
const handleClick = useCallback(() => setOpen(true), [setOpen]);

// useCallback internal:
// 1. Hook list'da slot olish (linked list traversal)
// 2. Deps comparison (Object.is on each)
// 3. Cache hit/miss
// 4. Function reference qaytarish

// Performance: deps array tekshiruv har render'da (kichik overhead)
```

Inline va `useCallback` — overhead taxminan teng. Real foyda faqat `React.memo` child'da bailout chaqirilganda.

**Memo bailout chain:**

```tsx
const Memo = memo(({ onClick }: { onClick: () => void }) => {
  console.log('Child render');
  return <button onClick={onClick}>X</button>;
});

function Parent() {
  // Inline:
  // return <Memo onClick={() => doSomething()} />;
  // → Har render yangi function → Memo shallowEqual fail → Child render har Parent render'da
  
  // useCallback:
  const handleClick = useCallback(() => doSomething(), []);
  return <Memo onClick={handleClick} />;
  // → Stable reference → Memo shallowEqual pass → Child re-render skip
}
```

`useCallback` foydasi — child render avoid qilish. Lekin Parent'ning o'zi har holda render qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Inline — sodda case:

```tsx
function ToggleButton() {
  const [open, setOpen] = useState(false);
  
  return (
    <Stack gap={4}>
      {/* Inline OK — sodda action */}
      <button onClick={() => setOpen(true)}>Open</button>
      <button onClick={() => setOpen(false)}>Close</button>
      <button onClick={() => setOpen((o) => !o)}>Toggle</button>
      
      {open && <p>Content</p>}
    </Stack>
  );
}
```

Separate — murakkab logic:

```tsx
function PaymentForm() {
  const [amount, setAmount] = useState(0);
  const [submitting, setSubmitting] = useState(false);
  const [error, setError] = useState<string | null>(null);
  
  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    setError(null);
    
    // Validation
    if (amount <= 0) {
      setError('Amount must be positive');
      return;
    }
    if (amount > 10000) {
      setError('Amount exceeds limit');
      return;
    }
    
    setSubmitting(true);
    
    try {
      const response = await fetch('/api/payment', {
        method: 'POST',
        body: JSON.stringify({ amount }),
      });
      
      if (!response.ok) {
        throw new Error('Payment failed');
      }
      
      const result = await response.json();
      console.log('Success:', result);
    } catch (err) {
      setError((err as Error).message);
    } finally {
      setSubmitting(false);
    }
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <Stack gap={8}>
        <input
          type="number"
          value={amount}
          onChange={(e) => setAmount(Number(e.target.value))}
        />
        {error && <p className="error">{error}</p>}
        <button type="submit" disabled={submitting}>
          {submitting ? 'Processing...' : 'Pay'}
        </button>
      </Stack>
    </form>
  );
}
```

Separate + useCallback (memo child):

```tsx
import { useCallback, memo } from 'react';

const ExpensiveItem = memo(function ExpensiveItem({ item, onSelect }: ItemProps) {
  // Heavy render logic
  return <li onClick={() => onSelect(item.id)}>{item.name}</li>;
});

function VirtualList({ items }: { items: Item[] }) {
  const [selected, setSelected] = useState<number | null>(null);
  
  // ✅ useCallback — child memo bailout uchun stable reference
  const handleSelect = useCallback((id: number) => {
    setSelected(id);
  }, []);
  
  return (
    <ul>
      {items.map((item) => (
        <ExpensiveItem
          key={item.id}
          item={item}
          onSelect={handleSelect}
        />
      ))}
    </ul>
  );
}
```

</details>

---

## Propagation va Default Behavior

### Nazariya

DOM event'lar ikki muhim mexanizmga ega:

1. **Propagation (bubble)** — child'da yuz bergan event parent'larga ko'tariladi
2. **Default behavior** — browser'ning predefined harakat'i (link → navigate, form → submit)

**`stopPropagation` — bubble to'xtatish:**

```tsx
function NestedClicks() {
  return (
    <div onClick={() => console.log('outer')}>
      <button
        onClick={(e) => {
          e.stopPropagation();
          console.log('inner');
        }}
      >
        Click
      </button>
    </div>
  );
}

// Click button:
// "inner" (stopPropagation — outer chaqirilmaydi)
```

**`preventDefault` — browser default to'xtatish:**

```tsx
// Form submit — default page reload
<form onSubmit={(e) => { e.preventDefault(); /* custom */ }}>...</form>

// Link click — default navigation
<a href="/home" onClick={(e) => { e.preventDefault(); /* custom */ }}>Home</a>

// Right click — default context menu
<div onContextMenu={(e) => { e.preventDefault(); /* custom menu */ }}>...</div>
```

**Common default behaviors:**

| Element | Default behavior |
|---------|------------------|
| `<form>` submit | Page reload + GET/POST request |
| `<a href>` click | Navigate to href |
| `<input type="checkbox">` click | Toggle checked |
| `<input type="submit">` click | Form submit |
| Right-click | Context menu |
| Drag link | Native drag |
| Keyboard navigation | Tab focus |

**`stopPropagation` vs `preventDefault` — alohida:**

```tsx
const handleSubmit = (e: React.FormEvent) => {
  e.preventDefault();    // To'xtatadi: form submit (page reload)
  e.stopPropagation();   // To'xtatadi: bubble to ancestors
  
  // Faqat preventDefault → bubble davom etadi (parent handler chaqiriladi)
  // Faqat stopPropagation → form submit yuz beradi
};
```

**`stopImmediatePropagation` — native API:**

```tsx
// React'da yo'q — native event'da:
e.nativeEvent.stopImmediatePropagation();
// Boshqa handler'larni ham to'xtatadi (bir element'da multiple listener)
```

**Capture phase:**

DOM event ikki fazada o'tadi:

1. **Capture** — root → target (yuqoridan pastga)
2. **Bubble** — target → root (pastdan yuqoriga)

React default — bubble. Capture handler — `onClickCapture`, `onMouseDownCapture`:

```tsx
<div
  onClick={() => console.log('bubble')}
  onClickCapture={() => console.log('capture')}
>
  <button>Click</button>
</div>

// Click button:
// "capture" (root → target)
// "bubble" (target → root)
```

Capture — kam ishlatiladi, lekin advanced patterns'da foydali (e.g. global event interceptor).

**Passive event listeners:**

Modern DOM API — `addEventListener('scroll', handler, { passive: true })` — handler `preventDefault()` chaqirmasligini va'da qiladi, browser scroll smoothness'ni saqlaydi:

```tsx
// React DOM scroll event'larida default passive: true
<div onScroll={(e) => {
  // e.preventDefault();  // ❌ Ignore qilinadi (passive)
}}>...</div>

// Native API bilan non-passive:
useEffect(() => {
  const el = ref.current;
  el?.addEventListener('wheel', handler, { passive: false });
  return () => el?.removeEventListener('wheel', handler);
}, []);
```

<details>
<summary><strong>Under the Hood</strong></summary>

DOM event flow (W3C):

```
Phase 1 — Capture:
  Window → Document → ... → target.parentNode → target

Phase 2 — Target:
  target

Phase 3 — Bubble:
  target → target.parentNode → ... → Document → Window
```

Har phase'da React Fiber tree traversal:

```ts
// react-dom internal (soddalashtirilgan)
function dispatchEventForPlugins(domEventName, eventSystemFlags, nativeEvent) {
  const isCapturePhase = (eventSystemFlags & IS_CAPTURE_PHASE) !== 0;
  const targetFiber = getClosestInstanceFromNode(nativeEvent.target);
  
  // Build event listener chain
  const listeners: Listener[] = [];
  let fiber = targetFiber;
  
  while (fiber !== null) {
    const handler = fiber.memoizedProps[reactEventName];
    if (handler !== undefined) {
      listeners.push({ fiber, handler });
    }
    fiber = fiber.return;
  }
  
  // Capture: process from root → target (reverse listeners)
  // Bubble: process from target → root (listeners as is)
  
  if (isCapturePhase) listeners.reverse();
  
  for (const { fiber, handler } of listeners) {
    syntheticEvent.currentTarget = fiber.stateNode;
    handler(syntheticEvent);
    
    if (syntheticEvent.isPropagationStopped()) break;
  }
}
```

`stopPropagation` — listener chain'da break:

```ts
SyntheticEvent.prototype.stopPropagation = function() {
  this._stopPropagation = true;
  this.nativeEvent.stopPropagation();  // Native bilan ham
};

SyntheticEvent.prototype.isPropagationStopped = function() {
  return this._stopPropagation;
};
```

`preventDefault` — native event'ga qaytariladi:

```ts
SyntheticEvent.prototype.preventDefault = function() {
  this._defaultPrevented = true;
  this.nativeEvent.preventDefault();  // Browser default to'xtatish
};
```

**Passive event listeners — performance:**

Modern browser'larda scroll/wheel listener'lar default passive (chunki preventDefault scroll'ni to'sib qoladi). React `onScroll`, `onWheel`, `onTouchMove` — passive=true tomonidan o'rnatiladi:

```ts
// react-dom internal
const PASSIVE_EVENTS = new Set(['scroll', 'wheel', 'touchstart', 'touchmove']);

function listenToNativeEvent(eventName, target) {
  const passive = PASSIVE_EVENTS.has(eventName);
  target.addEventListener(eventName, dispatcher, { passive });
}
```

Non-passive kerak bo'lsa — `useEffect` + native `addEventListener`:

```tsx
useEffect(() => {
  const handler = (e: WheelEvent) => {
    e.preventDefault();  // Passive=false bo'lsa ishlaydi
  };
  document.addEventListener('wheel', handler, { passive: false });
  return () => document.removeEventListener('wheel', handler);
}, []);
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Modal — overlay click close:

```tsx
function Modal({ children, onClose }: { children: React.ReactNode; onClose: () => void }) {
  return (
    <div className="overlay" onClick={onClose}>
      <div
        className="modal"
        onClick={(e) => e.stopPropagation()}
      >
        {children}
      </div>
    </div>
  );
  // Click overlay → onClose
  // Click modal content → stopPropagation → onClose chaqirilmaydi
}
```

Form — preventDefault:

```tsx
function ContactForm() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    // ↑ Browser'ning default form submit (page reload) to'xtatiladi
    
    // Custom submit logic
    fetch('/api/contact', {
      method: 'POST',
      body: JSON.stringify({ name, email }),
    });
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <Stack gap={8}>
        <input value={name} onChange={(e) => setName(e.target.value)} />
        <input value={email} onChange={(e) => setEmail(e.target.value)} />
        <button type="submit">Send</button>
      </Stack>
    </form>
  );
}
```

Custom context menu:

```tsx
function CustomContextMenu() {
  const [menuPos, setMenuPos] = useState<{ x: number; y: number } | null>(null);
  
  const handleContextMenu = (e: React.MouseEvent) => {
    e.preventDefault();
    setMenuPos({ x: e.clientX, y: e.clientY });
  };
  
  const handleCloseMenu = () => setMenuPos(null);
  
  return (
    <div onContextMenu={handleContextMenu} onClick={handleCloseMenu}>
      <p>Right-click anywhere</p>
      
      {menuPos && (
        <ul
          style={{
            position: 'fixed',
            left: menuPos.x,
            top: menuPos.y,
            background: '#fff',
            border: '1px solid #ccc',
          }}
          onClick={(e) => e.stopPropagation()}
        >
          <li>Copy</li>
          <li>Paste</li>
          <li>Delete</li>
        </ul>
      )}
    </div>
  );
}
```

Capture handler — global keyboard intercept:

```tsx
function KeyboardShortcutsLayer({ children }: { children: React.ReactNode }) {
  const handleKeyDownCapture = (e: React.KeyboardEvent) => {
    if (e.ctrlKey && e.key === 's') {
      e.preventDefault();
      console.log('Save shortcut intercepted');
      // Capture phase — bubble'ga yetishmasdan handle
    }
  };
  
  return (
    <div onKeyDownCapture={handleKeyDownCapture}>
      {children}
    </div>
  );
}
```

</details>

---

## Common Events Catalog

### Nazariya

React'da eng ko'p ishlatiladigan event'lar va ularning qo'llanishi:

**Mouse events:**

```tsx
<button
  onClick={(e) => {}}            // Click
  onDoubleClick={(e) => {}}      // Double click
  onMouseDown={(e) => {}}        // Press
  onMouseUp={(e) => {}}          // Release
  onMouseEnter={(e) => {}}       // Enter (no bubble)
  onMouseLeave={(e) => {}}       // Leave (no bubble)
  onMouseOver={(e) => {}}        // Hover (bubbles)
  onMouseOut={(e) => {}}         // Out (bubbles)
  onMouseMove={(e) => {}}        // Move
/>
```

**Keyboard events:**

```tsx
<input
  onKeyDown={(e) => {}}    // Key press start (character va special keys ikkalasi)
  onKeyUp={(e) => {}}      // Key release
  onKeyPress={(e) => {}}   // ⚠️ Deprecated (MDN: deprecated, use 'keydown' yoki 'beforeinput')
/>
```

**Form events:**

```tsx
<form onSubmit={(e) => {}} onReset={(e) => {}}>
  <input
    onChange={(e) => {}}     // Value o'zgardi
    onInput={(e) => {}}       // Har character (real-time)
    onFocus={(e) => {}}       // Focus oldi
    onBlur={(e) => {}}        // Focus yo'qotdi
    onSelect={(e) => {}}      // Text select
    onInvalid={(e) => {}}     // Validation fail
  />
</form>
```

**Touch events:**

```tsx
<div
  onTouchStart={(e) => {}}    // Touch start
  onTouchMove={(e) => {}}     // Touch move
  onTouchEnd={(e) => {}}      // Touch end
  onTouchCancel={(e) => {}}   // Touch interrupted
/>
```

**Pointer events (modern, unified):**

```tsx
<div
  onPointerDown={(e) => {}}    // Mouse/touch/pen down
  onPointerMove={(e) => {}}    // Move
  onPointerUp={(e) => {}}      // Up
  onPointerEnter={(e) => {}}   // Enter
  onPointerLeave={(e) => {}}   // Leave
  onPointerCancel={(e) => {}}  // Cancel
/>
```

Pointer events — modern unified API (mouse + touch + pen). Mobile-friendly.

**Drag events:**

```tsx
<div
  draggable
  onDragStart={(e) => {}}      // Drag boshlandi
  onDrag={(e) => {}}            // Drag davom
  onDragEnd={(e) => {}}         // Drag tugadi
  onDragEnter={(e) => {}}       // Drag enter zone
  onDragOver={(e) => { e.preventDefault(); }}  // Drag over (preventDefault — drop zone make)
  onDragLeave={(e) => {}}       // Drag leave
  onDrop={(e) => {}}            // Drop
/>
```

**Clipboard events:**

```tsx
<input
  onCopy={(e) => {}}     // Ctrl+C
  onCut={(e) => {}}      // Ctrl+X
  onPaste={(e) => {}}    // Ctrl+V
/>
```

**Animation va Transition events:**

```tsx
<div
  className="animated"
  onAnimationStart={(e) => {}}
  onAnimationEnd={(e) => {}}
  onAnimationIteration={(e) => {}}
  onTransitionEnd={(e) => {}}
/>
```

**Wheel event:**

```tsx
<div onWheel={(e) => { console.log(e.deltaY); }}>...</div>
// deltaY > 0 — pastga, < 0 — yuqoriga
```

**Scroll event:**

```tsx
<div onScroll={(e) => { console.log(e.currentTarget.scrollTop); }}>...</div>
// Passive (default) — preventDefault ignore qilinadi
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Native event mapping:**

```ts
// react-dom internal — soddalashtirilgan
const REACT_TO_DOM = {
  onClick: 'click',
  onDoubleClick: 'dblclick',
  onMouseDown: 'mousedown',
  onMouseUp: 'mouseup',
  // mouseenter/mouseleave non-bubbling — React mouseover/mouseout'ni delegation'da
  // ushlaydi, keyin relatedTarget orqali enter/leave'ni synthesize qiladi:
  onMouseEnter: 'mouseover',  // synthesized (relatedTarget ancestor check)
  onMouseLeave: 'mouseout',   // synthesized
  onKeyDown: 'keydown',
  onKeyUp: 'keyup',
  // React'ning onChange'i text input'lar uchun native 'input' event'ni ushlaydi
  // (har character'da fire). Native 'change' event text input'da blur'gacha fire
  // bo'lmaydi. Checkbox/radio/select uchun native 'change' va React onChange
  // semantikasi taxminan bir xil (darhol fire). React tekstli input'lar uchun
  // convenience semantikasi qo'shadi.
  onChange: 'input',  // text input'lar uchun (chekbox/select uchun 'change')
  // ...
};
```

**onChange vs onInput:**

Text input'larda React `onChange` — value har o'zgarishda chaqiriladi (native `input` event semantikasi). Native HTML `onchange` esa text input'da focus yo'qotgandan keyin (blur) chaqiriladi. Checkbox, radio va select uchun ikkalasi ham darhol fire bo'ladi — bu yerda farq yo'q.

```tsx
<input
  type="text"
  onChange={(e) => console.log(e.target.value)}
  // React: har character'da chaqiriladi
  // Native onchange: blur'gacha chaqirilmaydi
/>
```

Bu — React'ning convenience improvement'i. Native `onInput` semantikasini React `onChange`'ga assign qildi.

**onMouseEnter/Leave bubbling:**

Native `mouseenter`/`mouseleave` — **non-bubbling** (DOM specification). Event delegation faqat bubble qiluvchi event'lar ishlatadi, shu sababli React `onMouseEnter`/`onMouseLeave` uchun bubble qiluvchi `mouseover`/`mouseout` event'larini root container'da ushlaydi va `relatedTarget` orqali enter/leave semantikasini simulyatsiya qiladi:

```ts
function syntheticMouseEnter(e: MouseEvent) {
  // mouseover bubble qiladi va target'ga keladi
  // relatedTarget — kelgan element (oldingi hover)
  // currentTarget.contains(relatedTarget) === false bo'lsa, hover yangi element'ga kirdi
  if (!e.currentTarget.contains(e.relatedTarget as Node)) {
    // onMouseEnter trigger
  }
}
```

**Pointer Events vs Mouse + Touch:**

Pointer events — W3C standart, modern browser'lar qo'llab-quvvatlaydi (Chrome 55+, Firefox 59+, Safari 13+). Eski browser'larda fallback:

```tsx
function UniversalDrag() {
  if ('PointerEvent' in window) {
    // Modern: onPointerMove
  } else {
    // Fallback: onMouseMove + onTouchMove
  }
}
```

React'da pointer event handler'lar (`onPointerDown`, `onPointerMove`, va h.k.) `react-dom` paketida boshidan qo'llab-quvvatlanadi — versiya-specific feature emas, browser support'ga bog'liq.

**`onChange` controlled vs uncontrolled:**

```tsx
// Controlled
<input value={value} onChange={(e) => setValue(e.target.value)} />

// Uncontrolled
<input defaultValue="initial" onChange={(e) => console.log(e.target.value)} />
```

Cross-ref [`14-lifting-and-controlled.md`](14-lifting-and-controlled.md).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Keyboard shortcuts:

```tsx
function KeyboardShortcuts() {
  const handleKeyDown = (e: React.KeyboardEvent<HTMLDivElement>) => {
    if (e.ctrlKey || e.metaKey) {
      switch (e.key) {
        case 's':
          e.preventDefault();
          save();
          break;
        case 'z':
          e.preventDefault();
          if (e.shiftKey) redo();
          else undo();
          break;
        case 'y':
          e.preventDefault();
          redo();
          break;
      }
    }
    
    if (e.key === 'Escape') {
      cancel();
    }
  };
  
  return (
    <div tabIndex={0} onKeyDown={handleKeyDown}>
      Press Ctrl+S to save, Ctrl+Z to undo, Esc to cancel
    </div>
  );
}
```

Drag and drop:

```tsx
type Item = { id: number; text: string };

function DragDropList({ items }: { items: Item[] }) {
  const [draggedId, setDraggedId] = useState<number | null>(null);
  const [list, setList] = useState(items);
  
  const handleDragStart = (id: number) => (e: React.DragEvent) => {
    setDraggedId(id);
    e.dataTransfer.effectAllowed = 'move';
  };
  
  const handleDragOver = (e: React.DragEvent) => {
    e.preventDefault();  // Allow drop
    e.dataTransfer.dropEffect = 'move';
  };
  
  const handleDrop = (targetId: number) => (e: React.DragEvent) => {
    e.preventDefault();
    if (draggedId === null || draggedId === targetId) return;
    
    setList((prev) => {
      const fromIdx = prev.findIndex((i) => i.id === draggedId);
      const toIdx = prev.findIndex((i) => i.id === targetId);
      const next = [...prev];
      const [removed] = next.splice(fromIdx, 1);
      next.splice(toIdx, 0, removed);
      return next;
    });
    setDraggedId(null);
  };
  
  return (
    <ul>
      {list.map((item) => (
        <li
          key={item.id}
          draggable
          onDragStart={handleDragStart(item.id)}
          onDragOver={handleDragOver}
          onDrop={handleDrop(item.id)}
          style={{ opacity: draggedId === item.id ? 0.5 : 1 }}
        >
          {item.text}
        </li>
      ))}
    </ul>
  );
}
```

Clipboard handling:

```tsx
function ClipboardArea() {
  const handlePaste = (e: React.ClipboardEvent<HTMLTextAreaElement>) => {
    const text = e.clipboardData.getData('text');
    console.log('Pasted:', text);
    
    // Custom logic — e.g. parse markdown
    if (text.startsWith('http')) {
      e.preventDefault();
      // Insert as link
    }
  };
  
  return <textarea onPaste={handlePaste} />;
}
```

Pointer events — universal:

```tsx
function PointerCanvas() {
  const [strokes, setStrokes] = useState<{ x: number; y: number }[][]>([]);
  const [drawing, setDrawing] = useState(false);
  
  const handlePointerDown = (e: React.PointerEvent<HTMLCanvasElement>) => {
    setDrawing(true);
    setStrokes((prev) => [...prev, [{ x: e.clientX, y: e.clientY }]]);
    e.currentTarget.setPointerCapture(e.pointerId);
    // ↑ Capture — pointer canvas'dan tashqariga chiqsa ham track qiladi
  };
  
  const handlePointerMove = (e: React.PointerEvent<HTMLCanvasElement>) => {
    if (!drawing) return;
    setStrokes((prev) => {
      const next = [...prev];
      next[next.length - 1] = [
        ...next[next.length - 1],
        { x: e.clientX, y: e.clientY },
      ];
      return next;
    });
  };
  
  const handlePointerUp = (e: React.PointerEvent<HTMLCanvasElement>) => {
    setDrawing(false);
    e.currentTarget.releasePointerCapture(e.pointerId);
  };
  
  return (
    <canvas
      width={800}
      height={600}
      onPointerDown={handlePointerDown}
      onPointerMove={handlePointerMove}
      onPointerUp={handlePointerUp}
      style={{ touchAction: 'none' }}  // Mobile scroll oldini olish
    />
  );
}
```

</details>

---

## R19 `<form action={fn}>` — Client-Side Actions

### Nazariya

R19'da `<form>` element'i yangi `action` prop bilan keladi: function uzatilsa, form submit'ni avtomatik handle qiladi.

> **🕐 Versiya evolyutsiyasi (`<form action>`):**
> - **Pre-R19:** `<form onSubmit>` — manual `e.preventDefault()`, FormData manual yig'ish.
> - **R19+:** `<form action={fn}>` — function avtomatik submit handler. `e.preventDefault()` shart emas. FormData object argument sifatida.
> - **Sabab:** Server Components (RSC) bilan birgalikda Server Actions integration. Client-side ham yangi API — progressive enhancement (JS off bo'lsa, native form submit).

```tsx
// Pre-R19 — onSubmit
function OldForm() {
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    // ... submit logic
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input name="email" type="email" />
      <button type="submit">Submit</button>
    </form>
  );
}

// R19+ — action prop
function NewForm() {
  const submitAction = (formData: FormData) => {
    // formData avtomatik to'plangan
    const email = formData.get('email');
    console.log('Submit:', email);
  };
  
  return (
    <form action={submitAction}>
      <input name="email" type="email" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

**Async action:**

Async function uzatish — pending state'ni `useFormStatus` bilan kuzatish mumkin:

```tsx
import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Submitting...' : 'Submit'}
    </button>
  );
}

function MyForm() {
  const submitAction = async (formData: FormData) => {
    const email = formData.get('email') as string;
    await fetch('/api/subscribe', {
      method: 'POST',
      body: JSON.stringify({ email }),
    });
  };
  
  return (
    <form action={submitAction}>
      <input name="email" type="email" required />
      <SubmitButton />
    </form>
  );
}
```

`useFormStatus` — eng yaqin parent `<form>`'dagi action'ning status'ini qaytaradi. **MUHIM**: hook `<form>` rendering qiluvchi bir xil component'da ishlamaydi — child component'da chaqirilishi shart (Form context provider'idan keyin tree'da bo'lishi kerak). Cross-ref [`23-r19-hooks.md`](23-r19-hooks.md).

**`useActionState` — action result:**

```tsx
import { useActionState } from 'react';

type State = { success: boolean; error?: string };

function MyForm() {
  const submitAction = async (prevState: State, formData: FormData): Promise<State> => {
    const email = formData.get('email') as string;
    
    if (!email) {
      return { success: false, error: 'Email required' };
    }
    
    try {
      await fetch('/api/subscribe', { method: 'POST', body: JSON.stringify({ email }) });
      return { success: true };
    } catch (err) {
      return { success: false, error: 'Failed to subscribe' };
    }
  };
  
  const [state, formAction, pending] = useActionState(submitAction, { success: false });
  
  return (
    <form action={formAction}>
      <input name="email" type="email" />
      <button type="submit" disabled={pending}>
        {pending ? '...' : 'Subscribe'}
      </button>
      {state.error && <p>{state.error}</p>}
      {state.success && <p>Subscribed!</p>}
    </form>
  );
}
```

Cross-ref [`23-r19-hooks.md`](23-r19-hooks.md) — useActionState batafsil.

**Server Actions integration:**

```tsx
// Server Component
'use server';
async function subscribeUser(formData: FormData) {
  const email = formData.get('email') as string;
  await db.subscribe(email);
}

// Client / Server Component
function NewsletterForm() {
  return (
    <form action={subscribeUser}>
      <input name="email" type="email" />
      <button type="submit">Subscribe</button>
    </form>
  );
  // Server'da execute qilinadi, no client-side fetch kerak
}
```

Cross-ref [`39-rsc-server-actions.md`](39-rsc-server-actions.md).

**Progressive enhancement:**

`<form action="/api/subscribe">` — string action default browser submit. JS off bo'lsa native form submit ishlaydi. JS on bo'lsa React function action ishlaydi:

```tsx
<form action={submitAction}>...</form>
// ↑ JS on: function chaqiriladi, JS off: native browser submit
```

**Reset on submit:**

R19'da `<form action={fn}>` action function tugagandan keyin (sync uchun darhol, async uchun resolve'dan keyin) `form.reset()` chaqiradi. Bu uncontrolled inputs'da form'ni avtomatik tozalaydi:

```tsx
const submitAction = (formData: FormData) => {
  console.log(formData.get('email'));
  // Action tugagach React form.reset() chaqiradi → uncontrolled inputs bo'shatiladi
};

return (
  <form action={submitAction}>
    <input name="email" type="email" />  {/* uncontrolled — reset bo'ladi */}
    <button type="submit">Send</button>
  </form>
);
```

Reset xohlamasangiz — controlled inputs ishlating (state React'da boshqariladi, `form.reset()` uncontrolled DOM input'larga ta'sir qiladi, controlled state'ga emas) yoki action ichida `requestFormReset` API'ni boshqaring.

**`formAction` button'da — multi-action form:**

R19'da `<button formAction={fn}>` / `<input type="submit" formAction={fn}>` — submit button'ning o'z action'i bo'lishi mumkin, form'ning `action` prop'idan farqli:

```tsx
function MultiActionForm() {
  const saveAction = async (formData: FormData) => {
    await fetch('/api/save', { method: 'POST', body: formData });
  };
  const publishAction = async (formData: FormData) => {
    await fetch('/api/publish', { method: 'POST', body: formData });
  };
  
  return (
    <form action={saveAction}>
      <input name="title" />
      <button type="submit">Save Draft</button>
      <button type="submit" formAction={publishAction}>Publish</button>
      {/* "Save Draft" → saveAction, "Publish" → publishAction */}
    </form>
  );
}
```

Bu — HTML5'ning `formaction` attribute'iga mos (native browser ham submit button'da `formaction="..."` URL ishlatishi mumkin). R19'da function ham qabul qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

R19 form action handling — internal:

```ts
// react-dom internal (soddalashtirilgan)
function handleFormSubmit(form: HTMLFormElement, action: string | ((fd: FormData) => unknown)) {
  if (typeof action === 'string') {
    // Native submit (URL'ga)
    return false;  // Don't preventDefault
  }
  
  if (typeof action === 'function') {
    // R19 client/server action
    const formData = new FormData(form);
    const result = action(formData);
    
    if (result instanceof Promise) {
      // Pending state — useFormStatus uchun
      setFormPending(form, true);
      result.finally(() => setFormPending(form, false));
    }
    
    // Reset uncontrolled fields
    form.reset();
    
    return true;  // preventDefault — native submit'ni to'xtatish
  }
}

// React internal listener
form.addEventListener('submit', (e) => {
  const action = props.action;
  if (typeof action === 'function') {
    e.preventDefault();
    handleFormSubmit(form, action);
  }
  // String action — native submit
});
```

`useFormStatus` — React Context (Form context):

```ts
const FormStatusContext = createContext({ pending: false, ... });

function useFormStatus() {
  return useContext(FormStatusContext);
}

// Form internal Provider:
<FormStatusContext.Provider value={{ pending, ... }}>
  {children}  {/* SubmitButton va boshqa form children */}
</FormStatusContext.Provider>
```

`useActionState` — internal pseudo-implementation (concurrent rendering optimization'lar bilan):

```ts
function useActionState<S, P>(
  action: (state: S, payload: P) => S | Promise<S>,
  initialState: S
): [S, (payload: P) => void, boolean] {
  const [state, setState] = useState(initialState);
  const [pending, setPending] = useState(false);
  
  const dispatch = useCallback((payload: P) => {
    setPending(true);
    // Action sync yoki async bo'lishi mumkin — Promise.resolve normalizatsiya
    Promise.resolve(action(state, payload))
      .then((next) => setState(next))
      .finally(() => setPending(false));
  }, [state, action]);
  
  return [state, dispatch, pending];
}

// Eslatma: real useActionState concurrent rendering uchun useTransition pattern'ini
// ichida ishlatadi (pending state interruptible bo'ladi). Form'da `<form action={dispatch}>`
// dispatch FormData'ni payload sifatida qabul qiladi.
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Sodda form action:

```tsx
function NewsletterForm() {
  const submitAction = async (formData: FormData) => {
    const email = formData.get('email') as string;
    
    if (!email) return;
    
    await fetch('/api/subscribe', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email }),
    });
  };
  
  return (
    <form action={submitAction}>
      <Stack gap={8}>
        <input name="email" type="email" placeholder="Email" required />
        <button type="submit">Subscribe</button>
      </Stack>
    </form>
  );
}
```

useFormStatus — pending button:

```tsx
import { useFormStatus } from 'react-dom';

function SubmitButton({ children }: { children: React.ReactNode }) {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Submitting...' : children}
    </button>
  );
}

function ContactForm() {
  const submitAction = async (formData: FormData) => {
    await new Promise((r) => setTimeout(r, 2000));  // Simulate
    console.log(Object.fromEntries(formData));
  };
  
  return (
    <form action={submitAction}>
      <Stack gap={8}>
        <input name="name" placeholder="Name" required />
        <input name="email" type="email" placeholder="Email" required />
        <textarea name="message" placeholder="Message" required />
        <SubmitButton>Send Message</SubmitButton>
      </Stack>
    </form>
  );
}
```

useActionState — full state management:

```tsx
import { useActionState } from 'react';

type FormState = {
  success: boolean;
  errors?: { [key: string]: string };
  message?: string;
};

function LoginForm() {
  const loginAction = async (
    prevState: FormState,
    formData: FormData
  ): Promise<FormState> => {
    const email = formData.get('email') as string;
    const password = formData.get('password') as string;
    
    const errors: { [key: string]: string } = {};
    if (!email) errors.email = 'Email required';
    if (!password) errors.password = 'Password required';
    if (password.length < 8) errors.password = 'Min 8 characters';
    
    if (Object.keys(errors).length > 0) {
      return { success: false, errors };
    }
    
    try {
      const response = await fetch('/api/login', {
        method: 'POST',
        body: JSON.stringify({ email, password }),
      });
      
      if (!response.ok) {
        return { success: false, message: 'Invalid credentials' };
      }
      
      return { success: true, message: 'Logged in!' };
    } catch (err) {
      return { success: false, message: 'Network error' };
    }
  };
  
  const [state, formAction, pending] = useActionState(loginAction, { success: false });
  
  return (
    <form action={formAction}>
      <Stack gap={8}>
        <div>
          <input name="email" type="email" placeholder="Email" />
          {state.errors?.email && <span className="error">{state.errors.email}</span>}
        </div>
        <div>
          <input name="password" type="password" placeholder="Password" />
          {state.errors?.password && <span className="error">{state.errors.password}</span>}
        </div>
        <button type="submit" disabled={pending}>
          {pending ? 'Logging in...' : 'Login'}
        </button>
        {state.message && (
          <p className={state.success ? 'success' : 'error'}>{state.message}</p>
        )}
      </Stack>
    </form>
  );
}
```

Progressive enhancement — string action fallback:

```tsx
function ProgressiveForm() {
  // Server-rendered HTML default action — native submit if JS fails
  return (
    <form action="/api/subscribe" method="POST">
      <input name="email" type="email" />
      <button type="submit">Subscribe</button>
    </form>
  );
}

// JS-enhanced — function action override
function EnhancedForm() {
  const submitAction = async (formData: FormData) => {
    // Custom client-side logic
    const email = formData.get('email');
    console.log('Enhanced submit:', email);
  };
  
  return (
    <form action={submitAction}>
      <input name="email" type="email" />
      <button type="submit">Subscribe</button>
    </form>
  );
}
```

</details>

---

## TypeScript Event Types

### Nazariya

React event'lari TypeScript'da **generic types** orqali typed. Element type bilan event property'lari aniq belgilanadi.

**Asosiy event tiplar:**

| Event | TypeScript type | Element |
|-------|-----------------|---------|
| Click | `React.MouseEvent<HTMLButtonElement>` | button, div, ... |
| Change | `React.ChangeEvent<HTMLInputElement>` | input, select, textarea |
| Submit | `React.FormEvent<HTMLFormElement>` | form |
| Focus | `React.FocusEvent<HTMLInputElement>` | input, textarea |
| KeyDown | `React.KeyboardEvent<HTMLInputElement>` | input, div, ... |
| Touch | `React.TouchEvent<HTMLDivElement>` | div, ... |
| Pointer | `React.PointerEvent<HTMLDivElement>` | div, canvas, ... |
| Drag | `React.DragEvent<HTMLDivElement>` | div, ... |
| Wheel | `React.WheelEvent<HTMLDivElement>` | div, ... |
| Animation | `React.AnimationEvent<HTMLDivElement>` | div, ... |
| Transition | `React.TransitionEvent<HTMLDivElement>` | div, ... |

**Generic E parameter — element type:**

```tsx
function handleClick(e: React.MouseEvent<HTMLButtonElement>) {
  e.currentTarget;  // HTMLButtonElement (type-safe)
  e.target;         // EventTarget (narrow kerak)
  e.clientX;        // number
}

function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
  e.currentTarget.value;  // string (input value)
  e.target.value;          // string (chunki ChangeEvent specific)
}
```

**Common signature patterns:**

```tsx
type ButtonHandler = (e: React.MouseEvent<HTMLButtonElement>) => void;
type InputHandler = (e: React.ChangeEvent<HTMLInputElement>) => void;
type FormHandler = (e: React.FormEvent<HTMLFormElement>) => void;
type KeyHandler<E extends HTMLElement> = (e: React.KeyboardEvent<E>) => void;
```

**Events without arguments:**

Komponent props'da event handler tip — odatda value-only signature:

```tsx
type Props = {
  onSelect: (id: number) => void;       // Custom event — value uzatish
  onClick: () => void;                   // Sodda — argumentsiz
  onSubmit: (data: FormData) => void;   // Form data
};
```

JSX inline'da React event signature'i:

```tsx
type Props = {
  onSelect: (id: number) => void;
};

function Item({ id, onSelect }: { id: number; onSelect: Props['onSelect'] }) {
  return (
    <button
      onClick={(e: React.MouseEvent<HTMLButtonElement>) => {
        // ↑ React event ham kerak bo'lsa, signature'da
        e.preventDefault();
        onSelect(id);
      }}
    >
      Select
    </button>
  );
}
```

**Event handler in props — implicit event:**

```tsx
type Props = {
  onClick: React.MouseEventHandler<HTMLButtonElement>;
  // ↑ MouseEventHandler<E> = (event: MouseEvent<E>) => void
  
  onChange: React.ChangeEventHandler<HTMLInputElement>;
};

function Button({ onClick }: Props) {
  return <button onClick={onClick}>Click</button>;
}

<Button onClick={(e) => console.log(e.clientX)} />
// ↑ TS infer: e: MouseEvent<HTMLButtonElement>
```

`React.MouseEventHandler<E>` — alias for `(e: MouseEvent<E>) => void`. Convention pattern.

**Conditional types:**

```tsx
type EventHandler<E extends keyof JSX.IntrinsicElements> = E extends 'button'
  ? React.MouseEventHandler<HTMLButtonElement>
  : E extends 'input'
    ? React.ChangeEventHandler<HTMLInputElement>
    : React.SyntheticEventHandler;
```

Advanced — JSX element name'ga qarab handler tip avtomatik aniqlanadi.

<details>
<summary><strong>Under the Hood</strong></summary>

`@types/react` event type hierarchy:

```ts
// react/index.d.ts (soddalashtirilgan)

interface SyntheticEvent<T = Element, E = Event> extends BaseSyntheticEvent<E, T, EventTarget> {}

interface UIEvent<T = Element, E = NativeUIEvent> extends SyntheticEvent<T, E> {
  detail: number;
  view: AbstractView;
}

interface MouseEvent<T = Element, E = NativeMouseEvent> extends UIEvent<T, E> {
  altKey: boolean;
  button: number;
  clientX: number;
  clientY: number;
  // ...
}

interface KeyboardEvent<T = Element> extends UIEvent<T, NativeKeyboardEvent> {
  key: string;
  code: string;
  shiftKey: boolean;
  ctrlKey: boolean;
  // ...
}

interface ChangeEvent<T = Element> extends SyntheticEvent<T> {
  target: EventTarget & T;  // ← T bilan typed
}

interface FormEvent<T = Element> extends SyntheticEvent<T> {}

interface FocusEvent<T = Element, R = Element> extends SyntheticEvent<T, NativeFocusEvent> {
  relatedTarget: (EventTarget & R) | null;
  target: EventTarget & T;
}
```

**`target` typing — event-specific:**

```ts
// ChangeEvent — target typed
e.target  // EventTarget & HTMLInputElement (chunki ChangeEvent specific)

// MouseEvent — target generic (target click manbai bo'lishi mumkin har element)
e.target  // EventTarget (narrow kerak)
```

**Event handler aliases:**

```ts
type EventHandler<E extends SyntheticEvent<any>> = { bivarianceHack(event: E): void }["bivarianceHack"];

type ReactEventHandler<T = Element> = EventHandler<SyntheticEvent<T>>;
type ClipboardEventHandler<T = Element> = EventHandler<ClipboardEvent<T>>;
type CompositionEventHandler<T = Element> = EventHandler<CompositionEvent<T>>;
type DragEventHandler<T = Element> = EventHandler<DragEvent<T>>;
type FocusEventHandler<T = Element> = EventHandler<FocusEvent<T>>;
type FormEventHandler<T = Element> = EventHandler<FormEvent<T>>;
type ChangeEventHandler<T = Element> = EventHandler<ChangeEvent<T>>;
type KeyboardEventHandler<T = Element> = EventHandler<KeyboardEvent<T>>;
type MouseEventHandler<T = Element> = EventHandler<MouseEvent<T>>;
// ...
```

**Bivariance hack:**

`bivarianceHack` — TypeScript'ning function parameter variance trick. Default'da function parameter contravariant, lekin React event handler'larda bivariant kerak (bir handler turli event tip'lariga to'g'ri kelishi uchun). Bu — TS internal implementation detail.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Type-safe event handlers:

```tsx
function TypedHandlers() {
  const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
    e.currentTarget.disabled = true;  // ✅ HTMLButtonElement
    e.clientX;                         // ✅ MouseEvent property
  };
  
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    e.target.value;        // ✅ string (input value)
    e.currentTarget.value; // ✅ string
  };
  
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    // ↑ HTMLFormElement
  };
  
  const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
    if (e.key === 'Enter') {
      e.currentTarget.blur();  // ✅ HTMLInputElement.blur()
    }
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input onChange={handleChange} onKeyDown={handleKeyDown} />
      <button onClick={handleClick}>Submit</button>
    </form>
  );
}
```

EventHandler aliases:

```tsx
type Props = {
  onClick: React.MouseEventHandler<HTMLButtonElement>;
  onChange: React.ChangeEventHandler<HTMLInputElement>;
  onSubmit: React.FormEventHandler<HTMLFormElement>;
};

function FormSection({ onClick, onChange, onSubmit }: Props) {
  return (
    <form onSubmit={onSubmit}>
      <input onChange={onChange} />
      <button onClick={onClick}>Submit</button>
    </form>
  );
}

// Usage:
<FormSection
  onSubmit={(e) => { /* e: FormEvent<HTMLFormElement> inferred */ }}
  onChange={(e) => { /* e: ChangeEvent<HTMLInputElement> inferred */ }}
  onClick={(e) => { /* e: MouseEvent<HTMLButtonElement> inferred */ }}
/>
```

Custom event prop signature:

```tsx
type ListProps<T extends { id: string | number }> = {
  items: T[];
  onItemClick: (item: T, e: React.MouseEvent<HTMLLIElement>) => void;
  // ↑ Domain item + native event
};

function List<T extends { id: string | number }>({ items, onItemClick }: ListProps<T>) {
  return (
    <ul>
      {items.map((item) => (
        <li
          key={item.id}
          onClick={(e) => onItemClick(item, e)}
        >
          {String(item.id)}
        </li>
      ))}
    </ul>
  );
}

type User = { id: number; name: string };

<List<User>
  items={users}
  onItemClick={(user, e) => {
    console.log('User:', user.name);  // ✅ user: User
    console.log('Click:', e.clientX);  // ✅ e: MouseEvent
  }}
/>
```

Generic event handler:

```tsx
function GenericKeyHandler<E extends HTMLElement>(
  e: React.KeyboardEvent<E>,
  callback: (key: string) => void
) {
  if (e.key === 'Enter') {
    callback(e.key);
  }
}

function App() {
  return (
    <input
      onKeyDown={(e) => GenericKeyHandler(e, (k) => console.log(k))}
    />
  );
}
```

</details>

---

## TypeScript Event Narrowing va Generic Handlers

### Nazariya

`event.target` tipi — `EventTarget` (generic). Specific element kerak bo'lsa — **narrowing** patterns.

**`instanceof` narrowing:**

```tsx
function handleClick(e: React.MouseEvent<HTMLDivElement>) {
  if (e.target instanceof HTMLButtonElement) {
    e.target.disabled = true;  // ✅ HTMLButtonElement
  }
  if (e.target instanceof HTMLInputElement) {
    console.log(e.target.value);  // ✅ HTMLInputElement
  }
}
```

`e.target` — bosilgan element, `e.currentTarget` — handler attach qilgan element. Delegated handler'da target turli elementlar bo'lishi mumkin.

**Type assertion (cast):**

```tsx
function handleClick(e: React.MouseEvent<HTMLDivElement>) {
  const target = e.target as HTMLButtonElement;
  target.disabled = true;
  // ⚠️ Cast — runtime check yo'q. Faqat ishonchli bo'lganda
}
```

`as` cast — TS'ga "ishonaman" deyish. Runtime safety yo'q. `instanceof` afzal.

**`closest()` narrowing:**

```tsx
function handleClick(e: React.MouseEvent<HTMLDivElement>) {
  const target = e.target as HTMLElement;
  const button = target.closest('button');  // Closest button ancestor
  if (button) {
    const id = button.dataset.id;
    console.log('Button clicked:', id);
  }
}
```

`closest(selector)` — Element method, target'dan yuqoriga selector bilan eshlashtiradi. Delegated handler'da nested element click — closest target topish.

**Generic event handlers:**

```tsx
function handleAnyKey<E extends HTMLElement>(e: React.KeyboardEvent<E>) {
  if (e.key === 'Enter') {
    e.currentTarget.blur();  // ✅ E.blur() (HTMLElement method)
  }
}

<input onKeyDown={handleAnyKey} />     // E = HTMLInputElement
<textarea onKeyDown={handleAnyKey} />  // E = HTMLTextAreaElement
```

**Type predicate function:**

```tsx
function isInput(target: EventTarget): target is HTMLInputElement {
  return target instanceof HTMLInputElement;
}

function handleChange(e: React.ChangeEvent<HTMLDivElement>) {
  if (isInput(e.target)) {
    console.log(e.target.value);  // ✅ HTMLInputElement
  }
}
```

`target is HTMLInputElement` — type predicate. Function return value bo'yicha TS narrow qiladi.

**Discriminated union — handler args:**

```tsx
type Action =
  | { type: 'click'; payload: { id: number } }
  | { type: 'select'; payload: { value: string } }
  | { type: 'delete'; payload: { id: number } };

function handleAction(action: Action) {
  switch (action.type) {
    case 'click':
      console.log('Click:', action.payload.id);  // payload: { id: number }
      break;
    case 'select':
      console.log('Select:', action.payload.value);  // payload: { value: string }
      break;
    case 'delete':
      console.log('Delete:', action.payload.id);
      break;
  }
}
```

Cross-ref [`10-props.md`](10-props.md) — Discriminated Unions.

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript narrowing — flow analysis:

```ts
type T = HTMLElement | null;
const target: T = ...;

if (target instanceof HTMLButtonElement) {
  // Bu nuqtada: target — HTMLButtonElement
  target.disabled = true;
} else {
  // Bu nuqtada: target — HTMLElement | null (HTMLButtonElement yo'q)
}
```

`instanceof` — TS'ning narrowing rule. Branch ichida tip torayadi.

**`unknown` vs `any`:**

```ts
const data: unknown = getData();
data.name;  // ❌ TS error — unknown'dan property kirib bo'lmaydi
if (typeof data === 'object' && data !== null && 'name' in data) {
  data.name;  // ✅ narrowed
}

const anyData: any = getData();
anyData.name;  // ✅ TS pass (lekin runtime'da xato bo'lishi mumkin)
```

`unknown` — typed `any`. Narrowing kerak — type-safe.

**Type predicate function:**

```ts
function isInput(target: EventTarget): target is HTMLInputElement {
  return target instanceof HTMLInputElement;
}

// TS internal:
// isInput true qaytarsa, target — HTMLInputElement deb sanaladi
```

`X is Y` — predicate, function return value true bo'lsa X tipini Y deb biladi.

**Generic event handler types:**

```ts
type GenericHandler<E extends HTMLElement, Ev extends Event = Event> = (
  e: React.SyntheticEvent<E, Ev>
) => void;

function ButtonHandler(e: React.MouseEvent<HTMLButtonElement>) {}
const handler: GenericHandler<HTMLButtonElement, MouseEvent> = ButtonHandler;
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Delegated handler with narrowing:

```tsx
function ToolBar() {
  const handleClick = (e: React.MouseEvent<HTMLDivElement>) => {
    const target = e.target;
    
    if (target instanceof HTMLButtonElement) {
      const action = target.dataset.action;
      if (action) {
        console.log('Action:', action);
      }
    }
    
    if (target instanceof HTMLAnchorElement) {
      e.preventDefault();
      console.log('Link click:', target.href);
    }
  };
  
  return (
    <div onClick={handleClick}>
      <button data-action="save">Save</button>
      <button data-action="cancel">Cancel</button>
      <a href="/help">Help</a>
    </div>
  );
}
```

closest() pattern:

```tsx
function ItemListWithDelegated() {
  const handleListClick = (e: React.MouseEvent<HTMLUListElement>) => {
    const target = e.target as HTMLElement;
    const item = target.closest<HTMLLIElement>('li[data-id]');
    
    if (item) {
      const id = Number(item.dataset.id);
      console.log('Item clicked:', id);
    }
  };
  
  return (
    <ul onClick={handleListClick}>
      <li data-id="1">
        <span>Item 1</span>
        <button>Action</button>
      </li>
      <li data-id="2">
        <span>Item 2</span>
        <button>Action</button>
      </li>
    </ul>
  );
  // Click anywhere within <li> — closest('li[data-id]') topadi
}
```

Type predicate function:

```tsx
function isFormInput(target: EventTarget): target is HTMLInputElement | HTMLTextAreaElement | HTMLSelectElement {
  return (
    target instanceof HTMLInputElement ||
    target instanceof HTMLTextAreaElement ||
    target instanceof HTMLSelectElement
  );
}

function FormContainer() {
  const handleChange = (e: React.ChangeEvent<HTMLDivElement>) => {
    if (isFormInput(e.target)) {
      console.log('Field changed:', e.target.name, '=', e.target.value);
      // ✅ name va value mavjud (form input)
    }
  };
  
  return (
    <div onChange={handleChange}>
      <input name="email" />
      <textarea name="message" />
      <select name="country">
        <option value="us">US</option>
        <option value="uz">UZ</option>
      </select>
    </div>
  );
}
```

Generic key handler:

```tsx
function useEnterKeyHandler<E extends HTMLElement>(
  callback: () => void
): (e: React.KeyboardEvent<E>) => void {
  return (e: React.KeyboardEvent<E>) => {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault();
      callback();
    }
  };
}

function ChatInput() {
  const handleSend = () => console.log('Send message');
  const handleEnter = useEnterKeyHandler<HTMLTextAreaElement>(handleSend);
  
  return <textarea onKeyDown={handleEnter} placeholder="Press Enter to send" />;
  // Enter — send, Shift+Enter — newline
}
```

</details>

---

## Edge Cases va Gotchas

### Gotcha 1: `onChange` vs `onInput` — React vs Native

```tsx
// React 'onChange' — har character'da chaqiriladi (native onInput'ga teng)
<input onChange={(e) => console.log(e.target.value)} />

// Native 'onChange' — blur'gacha chaqirilmaydi
// <input onchange="..."> — focus yo'qotgandan keyin
```

React `onChange` semantikasi — convenience improvement (real-time updates). Native `onInput` API — ko'p case'larda kerak emas (chunki React `onChange` allaqachon real-time).

---

### Gotcha 2: `e.target` Bubble Path'da O'zgaradi

```tsx
function NestedClicks() {
  return (
    <div onClick={(e) => {
      console.log('div target:', e.target);  // Click manbai (button)
      console.log('div current:', e.currentTarget);  // div
    }}>
      <button onClick={(e) => {
        console.log('btn target:', e.target);  // Click manbai (button)
        console.log('btn current:', e.currentTarget);  // button
      }}>
        Click
      </button>
    </div>
  );
}

// Click button:
// btn target: <button>, btn current: <button>
// div target: <button>, div current: <div>
//
// `target` — har doim original (manbai), `currentTarget` — bubble path bo'ylab o'zgaradi
```

---

### Gotcha 3: Form `<input type="submit">` va `<button>`

```tsx
// Default button type — "submit" (form ichida bo'lsa)
<form onSubmit={handleSubmit}>
  <button>Click</button>  {/* Default type="submit" — form submit qiladi */}
  
  <button type="button">Click</button>  {/* Submit qilmaydi */}
</form>
```

Form ichidagi `<button>` — default `type="submit"`. Submit qilmasligi kerak bo'lsa — `type="button"` explicit yozish.

---

### Gotcha 4: Synthetic vs Native Event — `currentTarget` Async

```tsx
const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
  console.log('Sync currentTarget:', e.currentTarget);  // <button>
  
  setTimeout(() => {
    console.log('Async currentTarget:', e.currentTarget);
    // R17+ — null yoki stale (currentTarget mutation handler chaqiruvi davomida)
  }, 0);
};
```

`currentTarget` — handler chaqiruvi davomida mutate qilinadi (per Fiber traversal). Async kontekstda — snapshot pattern:

```tsx
const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
  const button = e.currentTarget;  // Snapshot
  setTimeout(() => {
    console.log(button);  // ✅ Saqlanadi
  }, 0);
};
```

---

### Gotcha 5: `onScroll` Passive — preventDefault Ignore

```tsx
<div onScroll={(e) => {
  e.preventDefault();  // ❌ Ignore qilinadi (passive default true)
}}>
  ...
</div>
```

R DOM `onScroll`, `onWheel`, `onTouchMove` — default passive. `preventDefault` ishlamaydi. Native `addEventListener('wheel', handler, { passive: false })` orqali workaround.

---

## Common Mistakes

### ❌ Xato 1: `onClick={fn()}` — Function Call

```tsx
// ❌ Function chaqirilgan, return value (undefined) onClick'ga
<button onClick={handleClick()}>Click</button>

// ✅ Function reference
<button onClick={handleClick}>Click</button>

// ✅ Wrapper bilan argument
<button onClick={() => handleClick(id)}>Click</button>
```

**Sabab:** `()` — function chaqiriladi (render paytida). React `undefined` event handler'ni e'tiborsiz qoldiradi → click hech narsa qilmaydi. Bonus muammo: render paytida side effect (handleClick body bajarildi).

---

### ❌ Xato 2: Form Submit'da `e.preventDefault()` Yo'q

```tsx
// ❌ Default submit yuz beradi (page reload)
const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
  // e.preventDefault() yo'q
  fetch('/api/submit', ...);  // Yuz beradi, lekin keyin page reload
};

// ✅
const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault();  // Native submit to'xtatiladi
  fetch('/api/submit', ...);
};

// ✅ R19 — action prop avtomatik
<form action={async (formData) => {
  await fetch('/api/submit', { method: 'POST', body: formData });
}}>
```

**Sabab:** Default form behavior — page reload. SPA'da bu xohlamagan. R19 `action` prop avtomatik handle qiladi.

---

### ❌ Xato 3: `onChange` Controlled Input'da Yo'q

```tsx
// ❌ Read-only input (warning console'da)
<input value={name} />

// ✅ Controlled (with onChange)
<input value={name} onChange={(e) => setName(e.target.value)} />

// ✅ Uncontrolled (defaultValue)
<input defaultValue="initial" />

// ✅ Read-only intentional
<input value={name} readOnly />
```

**Sabab:** `value` prop bilan input controlled bo'ladi. `onChange` yo'q — user typing'ga reaction yo'q (read-only). React warning beradi.

---

### ❌ Xato 4: `stopPropagation` Capture-Phase Listener'lar va Bir-Node Listener'larni To'xtatmaydi

```tsx
// Document'da global listener — CAPTURE phase
document.addEventListener('click', () => console.log('Global capture'), { capture: true });

const handleClick = (e: React.MouseEvent) => {
  e.stopPropagation();
  // ⚠️ Capture listener — React handler ishga tushishidan OLDIN chaqirilgan
  // stopPropagation kech keladi → "Global capture" chiqib bo'lgan
};
```

`e.stopPropagation()` SyntheticEvent ichida `nativeEvent.stopPropagation()` ham chaqiradi. Bubble phase'da React handler root container'da fire bo'ladi va undan keyingi bubble (document, window) to'xtatiladi — odatdagi `document.addEventListener('click', fn)` (bubble) **chaqirilmaydi**. Lekin ikki kategoriya hali ham ishlaydi:

1. **Capture-phase listener'lar** — React handler'gacha allaqachon chaqirilgan (capture target'gacha, keyin bubble root'gacha — order: document-capture → root-capture → target → root-bubble → React handler).
2. **React listener attach qilingan node'dagi boshqa listener'lar** — `nativeEvent.stopPropagation()` boshqa nodes'ga bubble qilmaydi, lekin **bir node'da registered listener'lar bir-birini to'xtatmaydi**.

```tsx
// ✅ Bir node'dagi boshqa listener'larni ham to'xtatish
const handleClick = (e: React.MouseEvent) => {
  e.nativeEvent.stopImmediatePropagation();
  // stopImmediatePropagation — bir node'dagi qolgan listener'lar ham skip
};
```

**Sabab:** `stopPropagation` faqat **keyingi node'larga** bubble'ni to'xtatadi. Capture-phase'da yoki bir node'dagi multiple listener uchun — `stopImmediatePropagation` kerak. Bundan tashqari, capture phase listener'lar har holda React handler'gacha chaqiriladi — bularni to'xtatish uchun capture-phase'da event'ni intercept qilish kerak.

---

### ❌ Xato 5: Inline Async Without Type

```tsx
// ❌ Type infer noaniq
<form onSubmit={async (e) => {
  e.preventDefault();
  await submit();
}}>

// ✅ Explicit type
<form onSubmit={async (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault();
  await submit();
}}>

// ✅✅ Best — separate handler
const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault();
  await submit();
};

return <form onSubmit={handleSubmit}>...</form>;
```

**Sabab:** TS event type inference inline'da ko'p hollarda OK, lekin async kontekstda explicit afzal (clarity, type-safe).

---

## Amaliy Mashqlar

### Mashq 1: Counter with Keyboard (Oson)

`Counter` komponent yarating: `+`/`-` button'lar va keyboard ↑/↓ keys'lar bilan ishlaydi.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);
  
  const handleKeyDown = (e: React.KeyboardEvent<HTMLDivElement>) => {
    if (e.key === 'ArrowUp') {
      setCount((c) => c + 1);
      e.preventDefault();
    }
    if (e.key === 'ArrowDown') {
      setCount((c) => c - 1);
      e.preventDefault();
    }
  };
  
  return (
    <div tabIndex={0} onKeyDown={handleKeyDown}>
      <p>Count: {count}</p>
      <Inline gap={4}>
        <button onClick={() => setCount((c) => c + 1)}>+</button>
        <button onClick={() => setCount((c) => c - 1)}>-</button>
      </Inline>
      <p>Or use ↑/↓ keys</p>
    </div>
  );
}
```

`tabIndex={0}` — div focusable bo'lishi uchun (default'da div focusable emas, `tabIndex` qo'shiladi). `onKeyDown` faqat focus bo'lganda ishlaydi.

</details>

---

### Mashq 2: Form Submit (Oson)

`ContactForm` yarating: `name` va `email` input, submit'da `e.preventDefault()`, FormData yig'ish, console.log.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useState } from 'react';

function ContactForm() {
  const [submitted, setSubmitted] = useState(false);
  
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    
    const formData = new FormData(e.currentTarget);
    const data = {
      name: formData.get('name') as string,
      email: formData.get('email') as string,
    };
    
    console.log('Submit:', data);
    setSubmitted(true);
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <Stack gap={8}>
        <input name="name" placeholder="Name" required />
        <input name="email" type="email" placeholder="Email" required />
        <button type="submit">Submit</button>
        {submitted && <p>Thanks!</p>}
      </Stack>
    </form>
  );
}

// R19 alternative:
function ContactFormR19() {
  const [submitted, setSubmitted] = useState(false);
  
  const submitAction = (formData: FormData) => {
    console.log('Submit:', {
      name: formData.get('name'),
      email: formData.get('email'),
    });
    setSubmitted(true);
  };
  
  return (
    <form action={submitAction}>
      <Stack gap={8}>
        <input name="name" placeholder="Name" required />
        <input name="email" type="email" placeholder="Email" required />
        <button type="submit">Submit</button>
        {submitted && <p>Thanks!</p>}
      </Stack>
    </form>
  );
}
```

R19 `action` prop avtomatik FormData uzatadi va `e.preventDefault()` shart emas.

</details>

---

### Mashq 3: Modal Click-Outside Close (O'rta)

`Modal` komponent yarating: overlay click'da close, modal content click'da close emas (stopPropagation), Escape key'da close.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useEffect } from 'react';

type ModalProps = {
  isOpen: boolean;
  onClose: () => void;
  children: React.ReactNode;
};

function Modal({ isOpen, onClose, children }: ModalProps) {
  // Escape key — global listener
  useEffect(() => {
    if (!isOpen) return;
    
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === 'Escape') {
        onClose();
      }
    };
    
    document.addEventListener('keydown', handleKeyDown);
    return () => document.removeEventListener('keydown', handleKeyDown);
  }, [isOpen, onClose]);
  
  if (!isOpen) return null;
  
  return (
    <div
      className="overlay"
      onClick={onClose}
      style={{
        position: 'fixed',
        inset: 0,
        background: 'rgba(0,0,0,0.5)',
      }}
    >
      <div
        className="modal"
        onClick={(e) => e.stopPropagation()}
        style={{
          background: 'white',
          padding: 16,
          margin: '10vh auto',
          maxWidth: 400,
        }}
      >
        {children}
      </div>
    </div>
  );
}

// Usage:
function App() {
  const [open, setOpen] = useState(false);
  
  return (
    <>
      <button onClick={() => setOpen(true)}>Open Modal</button>
      <Modal isOpen={open} onClose={() => setOpen(false)}>
        <h2>Modal Content</h2>
        <p>Click overlay or press Escape to close.</p>
        <button onClick={() => setOpen(false)}>Close</button>
      </Modal>
    </>
  );
}
```

**Asosiy nuqtalar:**

1. **Overlay onClick** → onClose
2. **Modal content onClick** → stopPropagation (overlay'ga bubble qilmasin)
3. **Escape key** — global listener (modal focus'siz ham ishlash uchun document'ga)
4. **Cleanup** — useEffect return — listener remove

</details>

---

### Mashq 4: Delegated Handler (O'rta)

`ToolBar` yarating: bitta `onClick` handler, ko'p button. Handler `e.target` orqali button'ni aniqlaydi va `data-action` attribute'ni o'qiydi.

<details>
<summary><strong>Javob</strong></summary>

```tsx
type ToolBarProps = {
  onAction: (action: string) => void;
};

function ToolBar({ onAction }: ToolBarProps) {
  const handleClick = (e: React.MouseEvent<HTMLDivElement>) => {
    const target = e.target;
    
    if (target instanceof HTMLButtonElement) {
      const action = target.dataset.action;
      if (action) {
        onAction(action);
      }
    }
  };
  
  return (
    <div className="toolbar" onClick={handleClick}>
      <button data-action="save">💾 Save</button>
      <button data-action="copy">📋 Copy</button>
      <button data-action="paste">📌 Paste</button>
      <button data-action="delete">🗑 Delete</button>
      <span>Some text (no action)</span>
    </div>
  );
}

// Usage:
function App() {
  const handleAction = (action: string) => {
    console.log('Action:', action);
  };
  
  return <ToolBar onAction={handleAction} />;
}

// Click "Save" → "Action: save"
// Click "Copy" → "Action: copy"
// Click "Some text" → No action (target HTMLSpanElement)
```

**Foyda:**

1. Bitta event handler — bir nechta button'lar uchun
2. Memory efficient — kam listener
3. Dynamic button qo'shish — handler kod o'zgartirishsiz ishlaydi

**`instanceof` narrowing** — type-safe access (`target.dataset.action`).

</details>

---

### Mashq 5: Custom Hook for Click-Outside (Qiyin)

`useClickOutside<T>` custom hook yarating: ref va callback qabul qiladi. Element tashqarisida click bo'lganda callback chaqiriladi.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useEffect, useRef } from 'react';

function useClickOutside<T extends HTMLElement>(
  callback: () => void
): React.RefObject<T> {
  const ref = useRef<T>(null);
  
  useEffect(() => {
    const handleClick = (e: MouseEvent) => {
      const target = e.target;
      if (
        ref.current &&
        target instanceof Node &&
        !ref.current.contains(target)
      ) {
        callback();
      }
    };
    
    // mousedown — click'dan oldin ishlaydi (better UX for menu close)
    document.addEventListener('mousedown', handleClick);
    return () => document.removeEventListener('mousedown', handleClick);
  }, [callback]);
  
  return ref;
}

// Usage:
function Dropdown() {
  const [isOpen, setIsOpen] = useState(false);
  const ref = useClickOutside<HTMLDivElement>(() => setIsOpen(false));
  
  return (
    <div ref={ref} style={{ position: 'relative' }}>
      <button onClick={() => setIsOpen((o) => !o)}>
        Menu {isOpen ? '▲' : '▼'}
      </button>
      
      {isOpen && (
        <ul style={{
          position: 'absolute',
          top: '100%',
          background: 'white',
          border: '1px solid #ccc',
        }}>
          <li>Item 1</li>
          <li>Item 2</li>
          <li>Item 3</li>
        </ul>
      )}
    </div>
  );
  // Click outside dropdown → onClickOutside → setIsOpen(false)
  // Click inside (button or list) → ref.current.contains(target) === true → no callback
}
```

**Asosiy nuqtalar:**

1. **Generic `<T extends HTMLElement>`** — ref har element type'iga moslashish (div, button, ...)
2. **Native `addEventListener('mousedown', ...)`** — document level (root container'dan tashqari)
3. **`ref.current.contains(target)`** — DOM API, target element ref ichida ekanligini tekshirish
4. **`mousedown` afzal `click`'dan** — UX: click'dan oldin trigger (open dropdown'da menu close, button click'idan oldin)
5. **Cleanup** — useEffect return

R17+ event delegation context: native `addEventListener('mousedown', ...)` ishlatiladi (chunki React `onClickOutside` yo'q, lekin global listener kerak). React `onMouseDown` faqat element ichidagi click'larni ushlaydi.

</details>

---

## Xulosa

- **Event handler** — JSX atribut camelCase, qiymat **function reference** (call qilinmagan)
- **SyntheticEvent** — cross-browser normalization wrapper, native event'ni o'rab oladi (`e.nativeEvent` original)
- **Event object** — `target` (click manbai), `currentTarget` (handler attach), `nativeEvent` (browser native)
- **Event delegation evolyutsiyasi** — R16 `document` → R17+ root container (microfrontends, multiple React apps)
- **Event pooling** — R17'da olib tashlandi, `e.persist()` deprecated (no-op)
- **R19 `<form action={fn}>`** — yangi client-side actions API, `useFormStatus`/`useActionState` integration, server actions cross-ref, progressive enhancement
- **Argument passing** — inline arrow wrapper (`() => handler(arg)`), naming convention `on<Action>` prop / `handle<Action>` handler
- **Inline vs separate handler** — sodda case'da inline OK, murakkab logic uchun separate (testability, useCallback bilan memo)
- **Propagation** — `stopPropagation` (bubble to'xtatish, R17+ native bilan ham), `preventDefault` (browser default to'xtatish)
- **Capture phase** — `onClickCapture` (root → target traversal), kam ishlatiladi
- **Common events** — Mouse, Keyboard, Form, Touch, Pointer (modern unified), Drag, Clipboard, Animation
- **TypeScript event types** — `MouseEvent<E>`, `ChangeEvent<E>`, `FormEvent<E>`, `KeyboardEvent<E>`, generic E element type
- **Event narrowing** — `instanceof` (runtime check), type predicate function (`target is HTMLInputElement`), `closest()` ancestor matching
- **Generic event handlers** — `<E extends HTMLElement>` element type generic

Keyingi bo'limda Lifting State up va Controlled vs Uncontrolled inputs yoritiladi: state ko'tarish (sibling components share data), controlled (React owns state) vs uncontrolled (DOM owns state) trade-off, hybrid pattern (uncontrolled + ref read on submit), va decision guide (qachon lift, qachon Context).

---

**Keyingi bo'lim:** [14-lifting-and-controlled.md](14-lifting-and-controlled.md) — Lifting state up (sibling sharing, single source of truth, inverse data flow), Controlled vs Uncontrolled inputs (`value` + `onChange` vs `defaultValue` + `useRef`), `defaultValue` semantikasi, common form patterns (input/textarea/select/checkbox/radio), hybrid pattern (uncontrolled + ref read on submit), decision guide (lift vs Context, controlled vs uncontrolled).
