# Bo'lim 7: JSX Asoslari va Qoidalari

> JSX — JavaScript'ga XML-style syntax qo'shadigan **language extension**. Babel/SWC compiler'lar tomonidan oddiy JS funksiya chaqiruvlariga aylantiriladi. JSX HTML emas — u JS, lekin tag-based syntax bilan. Bu bo'lim JSX va TSX farqi, JSX vs HTML qoidalari, transform mexanikasi (Classic vs Automatic R17+), expressions, fragments, conditional rendering va xavfsizlik nuanslari'ni yoritadi.

---

## Mundarija

- [JSX vs TSX Terminologiyasi](#jsx-vs-tsx-terminologiyasi)
- [JSX Nima](#jsx-nima)
- [JSX vs HTML Farqlari](#jsx-vs-html-farqlari)
- [JSX Transform — Classic vs Automatic](#jsx-transform--classic-vs-automatic)
- [JSX Expressions va Comments](#jsx-expressions-va-comments)
- [Single Root va Fragments](#single-root-va-fragments)
- [Reserved Attribute Names](#reserved-attribute-names)
- [Self-Closing va Boolean Attributes](#self-closing-va-boolean-attributes)
- [Spread Attributes](#spread-attributes)
- [HTML Insertion API va XSS](#html-insertion-api-va-xss)
- [Conditional Rendering](#conditional-rendering)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## JSX vs TSX Terminologiyasi

### Nazariya

**JSX** va **TSX** ko'pincha aralashtiriladi. Aniq farq:

| Terminologiya | Ma'no |
|---------------|-------|
| **JSX** | JavaScript XML — syntax extension nomi (Facebook 2013, JSX spec'i) |
| **TSX** | TypeScript JSX — `.tsx` fayl extension'i |
| **`.jsx` fayl** | JSX yozilgan JavaScript fayli |
| **`.tsx` fayl** | JSX yozilgan TypeScript fayli (turlar bilan) |

**Asosiy haqiqat:**

- **Syntax nomi har doim "JSX"** — kod TSX bo'lsa ham
- **JSX qoidalari TSX'da bir xil**
- **Runtime'da JSX va TSX farqi YO'Q** — compiler'dan keyin ikkalasi ham bir xil `_jsx(...)` chaqiruvlariga aylanadi
- **Farq faqat type system layer'da**

```jsx
// .jsx fayli
function Button({ label, onClick }) {
  return <button onClick={onClick}>{label}</button>;
}
```

```tsx
// .tsx fayli — turlar qo'shildi
interface ButtonProps {
  label: string;
  onClick: () => void;
}

function Button({ label, onClick }: ButtonProps) {
  return <button onClick={onClick}>{label}</button>;
}
```

Babel/SWC ikkala variantni ham bir xil `_jsx('button', { onClick, children: label })` chaqiruviga aylantiradi. TypeScript turlari **compile-time only** — production bundle'ga tushmaydi.

### Bu kursda

Bu kursda **TSX** majburiy. Sabab:

1. Production React kodbazalarining katta qismi TypeScript bilan yozilgan
2. Type safety — bug'larni compile-time'da topish
3. IDE autocomplete va refactoring yaxshi

---

## JSX Nima

### Nazariya

**JSX** — Facebook tomonidan 2013-yilda joriy etilgan **syntactic extension** for JavaScript. JSX **HTML emas**, lekin HTML'ga o'xshash syntax.

```tsx
// JSX
const element = <h1 className="title">Salom, dunyo!</h1>;

// Babel/SWC transform:
const element = _jsx('h1', { className: 'title', children: 'Salom, dunyo!' });
```

**Asosiy printsiplar:**

1. **JSX ham JS expression** — JS o'zgaruvchisiga assign qilish, return qilish, argument
2. **Tag — element type** — string yoki function/class
3. **Attributes — props obyekti**
4. **Children — props.children**

### JSX vs createElement

```tsx
// JSX
function Greeting() {
  return (
    <div className="greeting">
      <h1>Salom</h1>
      <p>React komponent</p>
    </div>
  );
}

// JSX'siz
function Greeting() {
  return React.createElement(
    'div',
    { className: 'greeting' },
    React.createElement('h1', null, 'Salom'),
    React.createElement('p', null, 'React komponent')
  );
}
```

JSX — **birinchi variant'ni yozish vositasi**.

<details>
<summary><strong>Under the Hood</strong></summary>

**JSX spec:** [https://facebook.github.io/jsx/](https://facebook.github.io/jsx/) — language-agnostic. React, Preact, Vue, Solid har biri o'z transformatsiyasi bilan ishlatadi.

**React Element structure** (JSX expression natijasi — `_jsx(...)` return qiymati):

```typescript
interface ReactElement {
  $$typeof: symbol;                // Symbol.for('react.element')
  type: string | ComponentType | symbol;  // 'div' | MyComp | REACT_FRAGMENT_TYPE
  key: string | null;              // JSX `key` attribute (props'dan ajratilgan)
  props: Record<string, unknown>;  // qolgan attribute'lar + children
  // ref: ko'p versiyalarda alohida slot edi; R19+ — boshqa prop'lar bilan birga props ichida
}
```

`$$typeof` — `Symbol.for('react.element')`. Bu — XSS protection marker: untrusted JSON serverdan kelganda React `JSON.parse` natijasidagi object'ni Element sifatida render qilmaydi, chunki `Symbol` JSON'da serializable emas (`JSON.parse(...)` natijasida `$$typeof` field `undefined` bo'ladi). Cross-realm `Symbol.for` orqali sun'iy `react.element` symbol yaratish uchun attacker'da allaqachon client-side JS execution kerak — ya'ni XSS allaqachon boshqa vector orqali sodir bo'lgan bo'lardi.

`type` qiymati uchun **uch variant** mavjud:
- `string` — DOM host (`'div'`, `'span'`)
- `function`/`class` — komponent (`MyButton`)
- `symbol` — maxsus React tip (`Fragment`, `Suspense`, `StrictMode`, `Profiler`)

**Component naming:**

- Lowercase `<div>` → string `_jsx('div', ...)`
- Capitalized `<MyComponent>` → reference `_jsx(MyComponent, ...)`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

JSX expression — har xil joylarda:

```tsx
// 1. O'zgaruvchiga assign
const heading = <h1 className="main">Welcome</h1>;

// 2. Komponent return
function Header() {
  return <h1>App</h1>;
}

// 3. Conditional
function Page({ user }: { user: User | null }) {
  return user ? <Dashboard /> : <Login />;
}

// 4. Array (lists)
const todos = [{ id: 1, text: 'Learn React' }];

const todoList = (
  <ul>
    {todos.map(todo => (
      <li key={todo.id}>{todo.text}</li>
    ))}
  </ul>
);

// 5. Function argument
function withBorder(content: ReactNode) {
  return <div className="bordered">{content}</div>;
}

const bordered = withBorder(<p>Inside border</p>);

// 6. Komponent prop
function Card({ header, body }: { header: ReactNode; body: ReactNode }) {
  return (
    <article>
      <header>{header}</header>
      <main>{body}</main>
    </article>
  );
}

<Card
  header={<h2>Card Title</h2>}
  body={<p>Card content</p>}
/>;
```

JSX = JS object:

```tsx
const list = <ul><li>Item</li></ul>;

console.log(typeof list);  // "object"
console.log(list.type);    // "ul"
console.log(list.props);   // { children: { type: 'li', props: {...} } }
```

</details>

---

## JSX vs HTML Farqlari

### Nazariya

JSX HTML'ga o'xshash, lekin **JavaScript reserved keyword'lar** va **camelCase convention** sababli farqlar bor:

### Asosiy farqlar jadvali

| HTML | JSX | Sabab |
|------|-----|-------|
| `class="primary"` | `className="primary"` | `class` — JS reserved keyword |
| `for="email"` | `htmlFor="email"` | `for` — JS reserved keyword |
| `tabindex="0"` | `tabIndex={0}` | camelCase |
| `readonly` | `readOnly` | camelCase |
| `maxlength="50"` | `maxLength={50}` | camelCase |
| `onclick="..."` | `onClick={...}` | camelCase + function reference |
| `style="color: red"` | `style={{ color: 'red' }}` | Object format |
| `<img src="...">` | `<img src="..." />` | Self-closing majburiy |
| `disabled` | `disabled` yoki `disabled={true}` | Boolean |

### `style` — string emas, object

```tsx
<div style={{ color: 'red', fontSize: '16px' }}>...</div>
```

**Diqqat qiling:**

- `style` — JavaScript object (**string TAQIQ**: `style="color: red"` React DOM dev mode'da error throws — `"The 'style' prop expects a mapping from style properties to values, not a string"`)
- Property nomlari camelCase (`fontSize`, not `font-size`)
- Number → ko'pchilik dimension property'lari uchun `px` avtomatik qo'shiladi, lekin **unitless property'lar bundan istisno**: `lineHeight`, `opacity`, `zIndex`, `flex`, `flexGrow`, `flexShrink`, `order`, `fontWeight`, `gridRow`, `gridColumn`, `columnCount`, `aspectRatio`, `tabSize`, `zoom`, SVG opacity property'lari (`fillOpacity`, `stopOpacity`, `strokeOpacity`) va boshqalar. To'liq ro'yxat React'ning `isUnitlessNumber` jadvalida.
- CSS custom properties (`--my-var`) — to'g'ridan-to'g'ri qo'llab-quvvatlanadi, `px` qo'shilmaydi: `style={{ '--accent': '#f00', color: 'var(--accent)' }}`
- Ikki marta `{{ }}` — birinchi JSX expression, ikkinchi object literal

```tsx
// ✅ Number — px (dimensional)
<div style={{ marginTop: 10 }}>...</div>
// Hosil: <div style="margin-top: 10px;">

// ✅ Unitless — px qo'shilmaydi
<div style={{ lineHeight: 1.5, opacity: 0.8, zIndex: 10 }}>...</div>
// Hosil: <div style="line-height: 1.5; opacity: 0.8; z-index: 10;">

// ✅ String — to'liq qiymat
<div style={{ marginTop: '10px' }}>...</div>

// ✅ Boshqa unit
<div style={{ width: '50%' }}>...</div>
```

### `onChange` — DOM bilan farqli

```tsx
<input value={value} onChange={(e) => setValue(e.target.value)} />
```

- **Native HTML/DOM:** `<input type="text">` va `<textarea>` uchun `change` event commit'da fires (blur yoki Enter); `<input type="checkbox|radio">`, `<select>` uchun darhol fires
- **React:** `onChange` text input/textarea uchun har keystroke'da fires (DOM `input` event'iga bog'langan); checkbox/radio/select uchun native bilan bir xil

```tsx
function Input() {
  return <input onChange={(e) => console.log(e.target.value)} />;
}
// User "a" yozdi → "a"
// User "b" yozdi → "ab"
// User "c" yozdi → "abc"
```

### Event handler — function reference

```tsx
// ❌ HTML inline string
<button onclick="handleClick()">Click</button>

// ✅ JSX function reference
<button onClick={handleClick}>Click</button>
```

`onClick={handleClick}` (reference) vs `onClick={handleClick()}` (call):

```tsx
// ✅ Reference
<button onClick={handleClick}>Click</button>

// ❌ Call — handleClick() darhol chaqiriladi
<button onClick={handleClick()}>Click</button>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**React DOM property mapping:**

```typescript
function setProperty(node, key, value) {
  if (key === 'className') {
    node.className = value;
  } else if (key === 'htmlFor') {
    node.htmlFor = value;
  } else if (key === 'style') {
    // CSS custom properties (--my-var) — setProperty; aks holda inline style
    for (const prop in value) {
      if (prop.startsWith('--')) node.style.setProperty(prop, value[prop]);
      else node.style[prop] = typeof value[prop] === 'number' && !isUnitlessNumber(prop)
        ? value[prop] + 'px'
        : value[prop];
    }
  } else if (key.startsWith('on')) {
    // React DOM event'ni element'ga emas — root container'ga
    // `listenToAllSupportedEvents(rootContainer)` orqali ulagan (R17+).
    // Element-level addEventListener YO'Q; React Synthetic Event delegation
    // event.target → fiber tree walk → handler dispatch.
  } else {
    node.setAttribute(key, value);
  }
}
```

**Reserved keyword'lar tarixi:**

JSX JS expression'ga transform qilinadi va attribute nomlari `_jsx(type, props, key)` chaqirig'idagi `props` object'ining key'lariga aylanadi. ES5 davrida `class` JS reserved word edi (ES3'dan), ES6'dan boshlab faqat strict mode'da reserved — lekin DOM API darajasida `Element.prototype.className` IDL property HTML element'larining standart property nomi. Shu sababli React `class` o'rniga `className` (DOM property nomi bilan mos), `for` o'rniga `htmlFor` (DOM `HTMLLabelElement.htmlFor`) tanladi. Standart HTML element'lar (`<div>`, `<label>`, ...) uchun `className`/`htmlFor` hozirgacha asosiy yo'l. **R19 yangiligi — Custom Elements support:** web component'lar (`<my-element>` kabi hyphen bilan lowercase tag'lar) uchun lowercase `class`, `for` va boshqa attribute'lar to'g'ridan-to'g'ri qabul qilinadi va element'ga setAttribute orqali uzatiladi. Bu standart HTML element'larga qo'llanilmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

JSX vs HTML to'liq misol:

```tsx
function ContactForm() {
  const [email, setEmail] = useState('');
  const [agreed, setAgreed] = useState(true);

  function handleSubmit(e: FormEvent<HTMLFormElement>) {
    e.preventDefault();
  }

  return (
    <form onSubmit={handleSubmit}>
      <label htmlFor="email">Email:</label>
      <input
        type="email"
        id="email"
        name="email"
        className="input primary"
        placeholder="example@example.com"
        required
        maxLength={100}
        autoComplete="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
      />
      <input
        type="checkbox"
        id="agree"
        checked={agreed}
        onChange={(e) => setAgreed(e.target.checked)}
      />
      <label htmlFor="agree">Agree to terms</label>
      <button type="submit" className="btn-primary">
        Submit
      </button>
    </form>
  );
}
```

`onChange` real-time validation:

```tsx
function EmailInput() {
  const [email, setEmail] = useState('');
  const [isValid, setIsValid] = useState(true);

  function handleChange(e: ChangeEvent<HTMLInputElement>) {
    const value = e.target.value;
    setEmail(value);
    setIsValid(/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value));
  }

  return (
    <div>
      <input
        type="email"
        value={email}
        onChange={handleChange}
        style={{ borderColor: isValid ? 'green' : 'red' }}
      />
      {!isValid && <p>Email noto'g'ri</p>}
    </div>
  );
}
```

</details>

---

## JSX Transform — Classic vs Automatic

### Nazariya

JSX kompilyator tomonidan JavaScript funksiya chaqiruvlariga aylantiriladi. Ikki turdagi transform:

### Classic Transform (R16 va undan oldin)

```tsx
// Manba kod:
import React from 'react';  // ← MAJBURIY

function App() {
  return <h1>Hello</h1>;
}

// Transform:
import React from 'react';

function App() {
  return React.createElement('h1', null, 'Hello');
}
```

### Automatic Transform (R17+)

```tsx
// Manba kod (no React import):
function App() {
  return <h1>Hello</h1>;
}

// Transform:
import { jsx as _jsx } from 'react/jsx-runtime';

function App() {
  return _jsx('h1', { children: 'Hello' });
}
```

> **🕐 Versiya evolyutsiyasi (JSX Transform):**
> - **Classic:** Pre-R17. `import React from 'react'` majburiy.
> - **Automatic (R17+):** Default. `import React` keraksiz.
> - **Sabab:** R17 release notes — bundle size optimization, modern tooling default.

### `_jsx`, `_jsxs`, `_jsxDEV` — uchta runtime entry-point

JSX Automatic Transform `children` shaklini va build mode'ni tahlil qilib, uch funksiyadan birini tanlaydi:

- **`_jsx(type, props, key?)`** — production build, `children` array EMAS (bitta element, string, number, yoki yo'q)
- **`_jsxs(type, props, key?)`** — production build, `children` **literal static array** (manba kodida bir nechta JSX child element yonma-yon yozilgan)
- **`_jsxDEV(type, props, key, isStaticChildren, source, self)`** — development build, har qanday holat uchun. Qo'shimcha argumentlar:
  - `isStaticChildren: boolean` — `_jsxs` equivalent (true) yoki `_jsx` equivalent (false)
  - `source: { fileName, lineNumber, columnNumber }` — error message'larda manba location
  - `self` — `this` context (legacy)

**Source paketlar:**
- `react/jsx-runtime` — `jsx`, `jsxs`, `Fragment` (production)
- `react/jsx-dev-runtime` — `jsxDEV`, `Fragment` (development)

**`_jsx` vs `_jsxs` farq sabab — key validation:** React dev mode'da children array element'larida `key` mavjudligini runtime'da tekshiradi. **Static children** (manbada `<A /><B />` yonma-yon yozilgan, transformer compile paytida ko'radi va array sifatida `children` ga joylaydi) — bu element'larning position'i bundle'da fixed, qayta tartiblanmaydi, shu sababli `key` kerak emas va `_jsxs` per-child key warning chiqarmaydi. **Dynamic array** holatida (`array.map(...)` natijasi) — array runtime'da hosil bo'ladi, transformer uning element'larini ko'rmaydi, `_jsx` ga **bitta `children` argument sifatida** array uzatiladi va React per-element `key` tekshiruvini yoqadi.

### Configuration

**Babel:**

```json
{
  "presets": [
    ["@babel/preset-react", { "runtime": "automatic" }]
  ]
}
```

**TypeScript:**

```json
{
  "compilerOptions": {
    "jsx": "react-jsx"
  }
}
```

`"jsx"` qiymatlari:
- `"react"` — Classic
- `"react-jsx"` — Automatic (R17+)
- `"react-jsxdev"` — Automatic + dev hints
- `"preserve"` — JSX'ni saqlash (Babel keyin)

<details>
<summary><strong>Under the Hood</strong></summary>

**`react/jsx-runtime` paketi:**

```typescript
// react/jsx-runtime exports:
export { jsx, jsxs } from './jsx-internal';
export { Fragment } from './ReactSymbols';

// react/jsx-dev-runtime — dev mode:
export { jsxDEV } from './jsx-dev-internal';
export { Fragment } from './ReactSymbols';
```

**`_jsxDEV` — qo'shimcha info:**

```typescript
// Production
_jsx('div', { children: 'Hello' });

// Development
_jsxDEV(
  'div',
  { children: 'Hello' },
  undefined,
  false,
  { fileName: '/src/App.tsx', lineNumber: 10, columnNumber: 5 },
  this
);
```

Dev mode'da error messages source location bilan keladi.

**Bundle size farqi:**

Classic transform — har faylda `import React`. Automatic — faqat `import { jsx } from 'react/jsx-runtime'`. Granular import, treeshake friendly.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Classic vs Automatic output:

```tsx
// Manba
function Greeting({ name }: { name: string }) {
  return (
    <div className="greeting">
      <h1>Hello, {name}</h1>
      <p>Welcome to React</p>
    </div>
  );
}
```

**Classic:**

```typescript
import React from 'react';

function Greeting({ name }) {
  return React.createElement(
    'div',
    { className: 'greeting' },
    React.createElement('h1', null, 'Hello, ', name),
    React.createElement('p', null, 'Welcome to React')
  );
}
```

**Automatic:**

```typescript
import { jsx as _jsx, jsxs as _jsxs } from 'react/jsx-runtime';

function Greeting({ name }) {
  return _jsxs('div', {
    className: 'greeting',
    children: [
      _jsxs('h1', { children: ['Hello, ', name] }),
      _jsx('p', { children: 'Welcome to React' }),
    ],
  });
}
```

Migration check:

```tsx
// ❌ R17+ da keraksiz
import React from 'react';
import { useState } from 'react';

// ✅ React import keraksiz
import { useState } from 'react';
```

**Eslatma:** Agar kodda `React.memo`, `React.Component` kabi namespace access ishlatilsa, `import React from 'react'` kerak. Lekin idiomatik pattern — `import { memo, Component, useState } from 'react'` named import — automatic transform bilan birga `React` default import'siz ishlaydi.

ESLint config R17+ uchun:

```json
{
  "rules": {
    "react/jsx-uses-react": "off",
    "react/react-in-jsx-scope": "off"
  }
}
```

</details>

---

## JSX Expressions va Comments

### Nazariya

JSX ichida JavaScript expression'lar **`{}`** ichida yoziladi:

```tsx
const name = 'Ali';
const age = 25;

<div>
  <p>Salom, {name}!</p>
  <p>Yosh: {age}</p>
  <p>Yosh + 5 = {age + 5}</p>
  <p>Function call: {greet(name)}</p>
  <p>Conditional: {age >= 18 ? 'Kattalar' : 'Bola'}</p>
</div>
```

**Qoidalar:**

1. **Expression — OK** — har qanday qiymatga eval bo'luvchi JS construct: arithmetic (`a + b`), function call (`fn()`), ternary (`a ? b : c`), logical (`a && b`, `a || b`, `a ?? b`), property access (`obj.x`), template literal, IIFE (`(() => {...})()`)
2. **Statement — TAQIQ**: `if`, `for`, `while`, `switch`, `try`, `var`/`let`/`const` declaration — chunki ular qiymat qaytarmaydi. Buning o'rniga ternary yoki IIFE ishlating
3. **Object literal — qo'sh `{{}}`** — birinchi `{` JSX expression delimiter, ikkinchi `{` object literal start

```tsx
// ✅ Expression
<p>{condition ? 'A' : 'B'}</p>
<p>{items.filter(x => x.active).map(x => x.name).join(', ')}</p>
<p>{(() => { const x = compute(); return x * 2; })()}</p>

// ❌ Statement — Syntax error
<p>{if (condition) 'A' else 'B'}</p>
<p>{for (let i = 0; i < 10; i++) <span>{i}</span>}</p>
```

### Render qilingan qiymatlar

| Qiymat | Render qiyoslash |
|--------|------------------|
| `string`, `number` | Text node |
| `bigint` | R18.3+: Text node (`String(bigint)` orqali); R18.2 va oldin: **Error** |
| `JSX element` | Nested element |
| `JSX array` | Lists (key kerak) |
| `null`, `undefined`, `false`, `true` | **Hech narsa** (skip) |
| `0` | **Text node "0"** (gotcha!) |
| `Promise` | R19+: to'g'ridan-to'g'ri JSX child sifatida qabul qilinadi — React Suspense boundary'gacha render'ni pauza qiladi, resolve bo'lganda continue (`use()` hook'i alohida pattern — komponent ichida any thenable'ni unwrap qiladi). R18 va undan oldin: **Error** |
| Plain `Object` (non-element) | **Error** "Objects are not valid as a React child" |

```tsx
<div>{null}</div>           // <div></div>
<div>{undefined}</div>      // <div></div>
<div>{false}</div>          // <div></div>
<div>{0}</div>              // <div>0</div>  ← GOTCHA
<div>{user}</div>           // ❌ Error: Objects are not valid

<div>{user.name}</div>      // ✅ Property access
<div>{JSON.stringify(user)}</div>  // ✅ String
```

### JSX Comments

```tsx
function App() {
  return (
    <div>
      {/* JSX comment */}
      <h1>App</h1>
      
      {/* Multi-line
          comment */}
      <p>Body</p>
    </div>
  );
}
```

**Qoidalar:**

- `{/* */}` — JSX expression sifatida JS comment
- `<!-- -->` — HTML comment **JSX'da ishlamaydi**

<details>
<summary><strong>Under the Hood</strong></summary>

**JSX expression — Babel transform:**

```tsx
<p>{count + 1}</p>

// Transform:
_jsx('p', { children: count + 1 });

// Reconciler type check:
function reconcileChildren(children) {
  if (typeof children === 'string' || typeof children === 'number') {
    return createTextFiber('' + children);
  }
  if (children == null || typeof children === 'boolean') {
    return null;
  }
  if (typeof children === 'object' && children.$$typeof === REACT_ELEMENT_TYPE) {
    return createFiberFromElement(children);
  }
  if (Array.isArray(children)) {
    return reconcileChildrenArray(children);
  }
  
  throw new Error('Objects are not valid as a React child');
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

JSX expression'lar:

```tsx
function UserCard({ user }: { user: User }) {
  return (
    <article className="user-card">
      <h2>{user.name}</h2>
      <p>Yosh: {user.age}</p>
      <p>Status: {user.isOnline ? 'Online' : 'Offline'}</p>
      <p>Joined: {formatDate(user.createdAt)}</p>
      {user.bio && <p>{user.bio}</p>}
      <p>{`Welcome, ${user.name}!`}</p>
      <ul>
        {user.hobbies.map(hobby => (
          <li key={hobby}>{hobby}</li>
        ))}
      </ul>
    </article>
  );
}
```

Object handling:

```tsx
const user = { name: 'Ali', age: 25 };

// ❌ Error
<div>{user}</div>

// ✅ Property access
<div>{user.name}</div>

// ✅ JSON.stringify (debug uchun)
<div><pre>{JSON.stringify(user, null, 2)}</pre></div>

// ✅ Multiple properties
<div>
  <p>Name: {user.name}</p>
  <p>Age: {user.age}</p>
</div>
```

Boolean rendering nuance — har variant alohida komponent sifatida:

```tsx
// 1. Boolean qiymat to'g'ridan-to'g'ri — render qilinmaydi
function StatusRaw({ isActive }: { isActive: boolean }) {
  return <p>Status: {isActive}</p>;
  // isActive=true:  <p>Status: </p>  ← "true" matni YO'Q (boolean skipped)
  // isActive=false: <p>Status: </p>
}

// 2. Explicit string conversion
function StatusString({ isActive }: { isActive: boolean }) {
  return <p>Status: {isActive.toString()}</p>;
  // isActive=true:  <p>Status: true</p>
  // isActive=false: <p>Status: false</p>
}

// 3. Ternary — production'da eng aniq variant
function StatusTernary({ isActive }: { isActive: boolean }) {
  return <p>Status: {isActive ? 'Yes' : 'No'}</p>;
  // isActive=true:  <p>Status: Yes</p>
}
```

</details>

---

## Single Root va Fragments

### Nazariya

JSX expression **bitta root element** qaytarishi shart.

```tsx
// ❌ Multiple roots — Syntax Error
function App() {
  return (
    <h1>Title</h1>
    <p>Body</p>
  );
}

// ✅ Wrapping div
function App() {
  return (
    <div>
      <h1>Title</h1>
      <p>Body</p>
    </div>
  );
}

// ✅ Fragment (extra DOM yo'q)
function App() {
  return (
    <>
      <h1>Title</h1>
      <p>Body</p>
    </>
  );
}
```

### Fragments

**`<></>`** — short syntax. **`<Fragment>`** — full syntax (key uchun).

```tsx
import { Fragment } from 'react';

// Short syntax
<>
  <Header />
  <Main />
</>

// Full syntax (key uchun)
<Fragment key="unique">
  <Header />
  <Main />
</Fragment>
```

**Fragments DOM'da hech narsa render qilmaydi.**

### Fragments va `key`

```tsx
{users.map(user => (
  <Fragment key={user.id}>
    <h3>{user.name}</h3>
    <p>{user.email}</p>
  </Fragment>
))}
```

Short syntax `<></>` key qabul qilmaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Fragment — React Symbol:**

```typescript
export const REACT_FRAGMENT_TYPE = Symbol.for('react.fragment');
```

JSX `<></>` Babel transform:

```tsx
<>
  <h1>A</h1>
  <p>B</p>
</>

// Automatic transform
_jsxs(_Fragment, {
  children: [
    _jsx('h1', { children: 'A' }),
    _jsx('p', { children: 'B' }),
  ],
});
```

**Reconciler:**

Fragment fiber **stateNode = null**. DOM mutation paytida Fragment skip qilinadi — child'lar bevosita parent host'ga `appendChild` qilinadi.

**`<></>` va `<Fragment>` farqi:** Ikkalasi ham `_jsxs(_Fragment, ...)` chaqirig'iga compile bo'ladi — bundle va Reconciler narxi bir xil. Farq syntax ergonomicssida: `<></>` qisqaroq, lekin `key` qabul qilmaydi; `<Fragment key="...">` to'liq syntax `key` ishlatilganda kerak. Ikkalasi ham DOM'da hech qanday element yaratmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Single root vs Fragment:

```tsx
// ❌ Multiple root — kompilatsiya xatosi
function ArticleHeaderBroken() {
  return (
    <h1>Title</h1>
    <p>Body</p>
  );
}

// ✅ Wrapping div (extra DOM)
function ArticleHeaderWithDiv() {
  return (
    <div>
      <h1>Title</h1>
      <p>Body</p>
    </div>
  );
}

// ✅ Fragment (no extra DOM)
function ArticleHeader() {
  return (
    <>
      <h1>Title</h1>
      <p>Body</p>
    </>
  );
}
```

Table list — Fragment kerak:

```tsx
function TableRows({ items }: { items: Item[] }) {
  return (
    <>
      {items.map(item => (
        <tr key={item.id}>
          <td>{item.name}</td>
          <td>{item.price}</td>
        </tr>
      ))}
    </>
  );
}

function ProductTable({ products }: { products: Product[] }) {
  return (
    <table>
      <thead>
        <tr><th>Name</th><th>Price</th></tr>
      </thead>
      <tbody>
        <TableRows items={products} />
      </tbody>
    </table>
  );
}
```

Fragment with key:

```tsx
import { Fragment } from 'react';

function DefinitionList({ entries }: { entries: { term: string; def: string }[] }) {
  return (
    <dl>
      {entries.map(entry => (
        <Fragment key={entry.term}>
          <dt>{entry.term}</dt>
          <dd>{entry.def}</dd>
        </Fragment>
      ))}
    </dl>
  );
}
```

Conditional Fragment:

```tsx
function ProfileCard({ user, showDetails }: ProfileCardProps) {
  return (
    <article>
      <h2>{user.name}</h2>
      {showDetails && (
        <>
          <p>Email: {user.email}</p>
          <p>Phone: {user.phone}</p>
        </>
      )}
    </article>
  );
}
```

</details>

---

## Reserved Attribute Names

### Nazariya

JSX'da ba'zi attribute nomlar HTML'dan farqli — JS reserved keyword'lar yoki convention bo'yicha camelCase'ga aylantirilgan.

### To'liq jadval

| HTML | JSX | Sabab |
|------|-----|-------|
| `class` | `className` | JS reserved keyword |
| `for` | `htmlFor` | JS reserved keyword |
| `tabindex` | `tabIndex` | camelCase |
| `readonly` | `readOnly` | camelCase |
| `maxlength` | `maxLength` | camelCase |
| `minlength` | `minLength` | camelCase |
| `cellspacing` | `cellSpacing` | camelCase |
| `cellpadding` | `cellPadding` | camelCase |
| `colspan` | `colSpan` | camelCase |
| `rowspan` | `rowSpan` | camelCase |
| `crossorigin` | `crossOrigin` | camelCase |
| `enctype` | `encType` | camelCase |
| `formaction` | `formAction` | camelCase |
| `srcset` | `srcSet` | camelCase |
| `usemap` | `useMap` | camelCase |
| `accesskey` | `accessKey` | camelCase |
| `contenteditable` | `contentEditable` | camelCase |
| `inputmode` | `inputMode` | camelCase |

### Event handler nomlari

Barcha event handler'lar **`on` + camelCase**:

| HTML | JSX |
|------|-----|
| `onclick` | `onClick` |
| `onchange` | `onChange` |
| `onsubmit` | `onSubmit` |
| `onfocus` | `onFocus` |
| `onblur` | `onBlur` |
| `onkeydown` | `onKeyDown` |
| `onkeyup` | `onKeyUp` |
| `onmouseenter` | `onMouseEnter` |
| `ontouchstart` | `onTouchStart` |
| `onpointerdown` | `onPointerDown` |

### `aria-*` va `data-*`

Bu attribute'lar **kebab-case** qoladi:

```tsx
<button
  aria-label="Close dialog"
  aria-pressed={isPressed}
  aria-describedby="tooltip"
  data-testid="close-button"
  data-product-id={product.id}
>
  ×
</button>
```

### SVG attribute'lar

```tsx
<svg viewBox="0 0 100 100">
  <circle
    cx={50}
    cy={50}
    r={40}
    fill="blue"
    strokeWidth={2}
    stroke="red"
  />
</svg>
```

| SVG | JSX |
|-----|-----|
| `stroke-width` | `strokeWidth` |
| `fill-opacity` | `fillOpacity` |
| `stop-color` | `stopColor` |
| `text-anchor` | `textAnchor` |
| `clip-path` | `clipPath` |

> **Eslatma:** TypeScript JSX types attribute nomlarini autocomplete qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`aria-*` va `data-*` exception:**

Bu attribute'lar JSX'da **kebab-case** qoladi:

```tsx
<button aria-label="Close" data-testid="btn">×</button>

// React DOM:
node.setAttribute('aria-label', 'Close');
node.setAttribute('data-testid', 'btn');
```

Sabab — HTML spec'i `aria-*` va `data-*` ni kebab-case sifatida belgilagan.

**Custom attribute'lar:**

R16+'dan boshlab JSX istalgan attribute qabul qiladi:

```tsx
<div my-custom-attr="value">...</div>
// React DOM: setAttribute('my-custom-attr', 'value')
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

To'liq form misol:

```tsx
function ContactForm() {
  return (
    <form
      action="/api/contact"
      method="POST"
      encType="multipart/form-data"
    >
      <label htmlFor="name">Name:</label>
      <input
        type="text"
        id="name"
        name="name"
        className="form-input"
        maxLength={100}
        required
        autoFocus
        autoComplete="name"
      />

      <label htmlFor="email">Email:</label>
      <input type="email" id="email" name="email" />

      <button
        type="submit"
        className="btn-primary"
        tabIndex={0}
        accessKey="s"
      >
        Submit
      </button>
    </form>
  );
}
```

Accessibility:

```tsx
function Modal({ isOpen, onClose, title, children }: ModalProps) {
  return (
    <div
      role="dialog"
      aria-labelledby="modal-title"
      aria-describedby="modal-content"
      aria-modal="true"
      aria-hidden={!isOpen}
      style={{ display: isOpen ? 'block' : 'none' }}
    >
      <h2 id="modal-title">{title}</h2>
      <div id="modal-content">{children}</div>
      <button onClick={onClose} aria-label="Close dialog">×</button>
    </div>
  );
}
```

SVG misol:

```tsx
function CircleIcon({ size = 24, color = 'currentColor' }: IconProps) {
  return (
    <svg
      width={size}
      height={size}
      viewBox="0 0 24 24"
      fill="none"
      xmlns="http://www.w3.org/2000/svg"
    >
      <circle
        cx={12}
        cy={12}
        r={10}
        stroke={color}
        strokeWidth={2}
        strokeLinecap="round"
      />
    </svg>
  );
}
```

</details>

---

## Self-Closing va Boolean Attributes

### Nazariya

### Self-Closing Tags

JSX'da **child'siz element'lar self-close** bo'lishi shart:

```tsx
// ✅ Self-closing
<img src="..." alt="..." />
<input type="text" />
<br />
<hr />

// ❌ HTML stilida — JSX syntax error
<img src="..." alt="...">
<input type="text">
```

### Boolean Attributes

```tsx
// ✅ Boolean qiymatlar
<input type="checkbox" checked={true} />
<input type="checkbox" checked={false} />
<button disabled={true}>Click</button>

// ✅ Shorthand (true bilan teng)
<input type="checkbox" checked />
<button disabled>Click</button>

// ✅ Conditional
<button disabled={isLoading}>
  {isLoading ? 'Loading...' : 'Submit'}
</button>
```

### Eng ko'p uchraydigan boolean attribute'lar

| Attribute | Qaerga |
|-----------|--------|
| `checked` | `<input type="checkbox/radio">` |
| `disabled` | `<button>`, `<input>`, `<select>`, `<textarea>` |
| `readOnly` | `<input>`, `<textarea>` |
| `required` | `<input>`, `<textarea>` |
| `autoFocus` | `<input>`, `<textarea>`, `<button>` |
| `multiple` | `<select>`, `<input type="file">` |
| `selected` | `<option>` |
| `loop` | `<audio>`, `<video>` |
| `muted` | `<audio>`, `<video>` |
| `autoPlay` | `<audio>`, `<video>` |
| `controls` | `<audio>`, `<video>` |
| `defer` | `<script>` |
| `async` | `<script>` |
| `open` | `<details>`, `<dialog>` |
| `hidden` | Universal |

<details>
<summary><strong>Under the Hood</strong></summary>

**Boolean attribute DOM mapping:**

```typescript
function setProperty(node, key, value) {
  if (typeof value === 'boolean') {
    if (value) {
      node.setAttribute(key, '');
    } else {
      node.removeAttribute(key);
    }
    return;
  }
  node.setAttribute(key, value);
}
```

**Some attribute'lar — special cases:**

```typescript
// `value` for inputs — DOM property
node.value = value;

// `checked` for checkboxes
node.checked = value;

// `selected` for options
node.selected = value;

// `htmlFor` for label
node.htmlFor = value;

// `className`
node.className = value;
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Boolean attribute'lar:

```tsx
function FormControls() {
  const [isLoading, setIsLoading] = useState(false);
  const [isAccepted, setIsAccepted] = useState(false);

  return (
    <form>
      <button type="submit" disabled={isLoading || !isAccepted}>
        {isLoading ? 'Loading...' : 'Submit'}
      </button>

      <input
        type="checkbox"
        checked={isAccepted}
        onChange={(e) => setIsAccepted(e.target.checked)}
      />

      <input type="text" value={generatedId} readOnly />
      <input type="email" required autoFocus autoComplete="email" />
      <input type="file" multiple accept="image/*" />
    </form>
  );
}
```

Self-closing components:

```tsx
function Layout() {
  return (
    <div>
      <Header />
      <Main>
        <Article />
      </Main>
      <Footer />
      
      <img src="/logo.svg" alt="Logo" width={120} height={40} />
      
      <hr />
    </div>
  );
}
```

</details>

---

## Spread Attributes

### Nazariya

**Spread attributes** — `{...props}` — obyekt'ning barcha property'larini element attribute sifatida pass qilish.

```tsx
const buttonProps = {
  className: 'btn',
  type: 'submit',
  disabled: false,
};

<button {...buttonProps}>Click</button>
// Equivalent:
<button className="btn" type="submit" disabled={false}>Click</button>
```

### Use cases

**1. Props forwarding:**

```tsx
interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary';
}

function Button({ variant = 'primary', className, ...rest }: ButtonProps) {
  return (
    <button
      className={`btn btn-${variant} ${className ?? ''}`}
      {...rest}
    />
  );
}

<Button variant="primary" onClick={handleClick} disabled={loading} type="submit">
  Save
</Button>
```

**2. Conditional props:**

```tsx
const inputProps = {
  type: 'text',
  ...(isReadOnly && { readOnly: true }),
  ...(maxLength && { maxLength }),
};

<input {...inputProps} />
```

**3. Rest pattern:**

```tsx
function Card({ title, ...rest }: CardProps) {
  return (
    <article {...rest}>
      <h2>{title}</h2>
    </article>
  );
}
```

### Xavflar

### 1. Unintended props forwarding

```tsx
// ❌ Custom prop DOM'ga
function Button({ isActive, ...rest }: { isActive: boolean }) {
  return <button {...rest} isActive={isActive} />;
  // <button isactive="true">  ← warning
}

// ✅ Filter qilish
function Button({ isActive, ...rest }: { isActive: boolean }) {
  return (
    <button
      {...rest}
      className={isActive ? 'active' : ''}
    />
  );
}
```

### 2. Override hazardlari

JSX attribute'lar **tartibda qo'llaniladi**:

```tsx
function Wrapper({ className, ...rest }: WrapperProps) {
  return (
    <div
      className="wrapper"  // ❌ Keyin override qiladi
      {...rest}
    />
  );
}

// ✅ Tartib to'g'ri + merge
function Wrapper({ className, ...rest }: WrapperProps) {
  return (
    <div
      {...rest}
      className={`wrapper ${className ?? ''}`}
    />
  );
}
```

### 3. Security — untrusted props

```tsx
// ❌ User input'ni spread
function UserCard({ userInput }: { userInput: any }) {
  return <div {...userInput}>...</div>;
}

// ✅ Aniq attribute'lar
function UserCard({ name, avatar }: UserCardProps) {
  return (
    <div className="user-card" data-user={name}>
      <img src={avatar} alt={name} />
    </div>
  );
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Babel transform:**

```tsx
<button {...rest} className="primary" disabled={loading}>
  Click
</button>

// Transform
_jsx('button', {
  ...rest,
  className: 'primary',
  disabled: loading,
  children: 'Click',
});
```

JS object spread — keyingi key oldingisini override qiladi.

**`key` va `ref` — spread'dan tashqari maxsus token'lar:**

JSX transform `key` va `ref` attribute'larini props object'idan **alohida ajratib oladi** va `_jsx(type, props, key)` chaqirig'idagi alohida positional argument sifatida uzatadi (yoki R19 da `ref` props ichiga qaytariladi — quyida). Shu sababli:

```tsx
<Comp {...obj} key="x" />
// Transform (automatic):
// _jsx(Comp, { ...obj }, "x")
//   ↑ key 3-argument, props ichida emas
```

Agar spread'dagi `obj` ichida `key` field bo'lsa, JSX transform uni 3-argument sifatida ajratmaydi (faqat explicit `key="..."` 3-argumentga aylanadi). `_jsx` runtime esa config object'ida `key` ko'rsa, uni `RESERVED_PROPS` ro'yxati bilan props'dan chiqarib, `element.key` ga ko'chiradi — ya'ni Reconciler aslida `key` ni oladi va ishlatadi. Lekin bu **brittle pattern**: agar keyingi render'da spread manbai (`obj.key`) yo'qolsa yoki o'zgarsa, element identity buziladi. Dev mode bu xavf uchun warning chiqaradi:

```
Warning: A props object containing a "key" prop is being spread into JSX:
  let props = { key: someKey, ...otherProps };
  <Component {...props} />
React keys must be passed directly to JSX without using spread:
  let props = otherProps;
  <Component key={someKey} {...props} />
```

**R19 `ref` o'zgarishi:** R18 va undan oldin `ref` ham `key` kabi alohida slot'da edi va functional component'da `forwardRef` orqali olinardi. **R19'dan boshlab `ref` oddiy prop sifatida props ichida** — `forwardRef` keraksiz, har qanday function component `ref` prop'ni to'g'ridan-to'g'ri qabul qiladi. Spread bilan `<Comp {...obj} ref={r} />` holatida `ref` props ichida (`props.ref = r`) qoladi.

**Performance:**

JSX compilationsi har render'da `_jsx(type, props, key)` chaqirig'i ichida yangi `props` object yaratadi (spread bo'lsa ham, bo'lmasa ham). Spread'ning o'zi qo'shimcha overhead emas — `Object.assign` equivalent tezligi. `React.memo` shallow compare qiymatlarni `Object.is` bilan tekshiradi: agar spread orqali uzatiladigan barcha qiymatlar (function reference, object reference) avvalgi render bilan teng bo'lsa, `memo` re-render qilmaydi. Lekin har render'da yangi inline object/function spread'ga tushsa — reference farq sababli memo bypass qilinadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Wrapping component:

```tsx
import { ButtonHTMLAttributes, Ref } from 'react';

interface CustomButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  isLoading?: boolean;
  ref?: Ref<HTMLButtonElement>;
}

// R19: ref oddiy prop sifatida — forwardRef keraksiz
function CustomButton({
  variant = 'primary',
  size = 'md',
  isLoading,
  className,
  children,
  disabled,
  ref,
  ...rest
}: CustomButtonProps) {
  return (
    <button
      ref={ref}
      className={`btn btn-${variant} btn-${size} ${className ?? ''}`}
      disabled={disabled || isLoading}
      {...rest}
    >
      {isLoading ? 'Loading...' : children}
    </button>
  );
}

<CustomButton
  variant="primary"
  size="lg"
  onClick={handleClick}
  type="submit"
  isLoading={loading}
  data-testid="save-btn"
  aria-label="Save form"
>
  Save
</CustomButton>
```

Override hazard fix:

```tsx
// ❌ Tartib noto'g'ri
function Bad({ className, ...rest }: Props) {
  return <div className="default-class" {...rest} />;
}

// ✅ Tartib to'g'ri + merge
function Good({ className, ...rest }: Props) {
  return (
    <div
      {...rest}
      className={`default-class ${className ?? ''}`}
    />
  );
}
```

</details>

---

## HTML Insertion API va XSS

### Nazariya

React'da DOM elementga **raw HTML insert qilish** uchun maxsus prop bor. Bu prop **`dangerously`** prefix'i bilan — chunki **XSS (Cross-Site Scripting)** xavfini keltiradi.

### Pattern

```tsx
// ⚠️ Trusted HTML faqat
<div dangerouslySetInnerHTML={{ __html: '<strong>Bold</strong>' }} />
```

**Format:**
- Prop nomi: `dangerouslySetInnerHTML`
- Qiymat: `{ __html: htmlString }` obyekt
- React `__html` key bilan obyekt'ni "intentional" deb hisoblaydi

### Nima uchun "dangerously"

React JSX standart usulda **HTML escape** qiladi:

```tsx
const userInput = '<script>alert("XSS")</script>';

<div>{userInput}</div>
// DOM: <div>&lt;script&gt;alert("XSS")&lt;/script&gt;</div>
// Text sifatida ko'rsatiladi. Xavfsiz.
```

`dangerouslySetInnerHTML` bu xavfsizlikni **chetlab o'tadi**:

```tsx
const userInput = '<script>alert("XSS")</script>';

<div dangerouslySetInnerHTML={{ __html: userInput }} />
// DOM: <div><script>alert("XSS")</script></div>
// Diqqat: <script> innerHTML orqali qo'shilganda execute QILINMAYDI (HTML5 parsing spec qoidasi).
// Lekin event handler attribute'lar (`onerror`, `onload`, `onclick`) va `javascript:` URL'lar
// orqali XSS hali mumkin — quyida real vector misol.
```

### Qachon ishlatiladi (legitimately)

1. **Server-rendered HTML** — markdown, rich text editor output
2. **Trusted CMS content**
3. **Third-party HTML widgets** — embed code
4. **Inline SVG content**
5. **Code highlighting** — Prism.js, highlight.js

### Sanitization

Untrusted HTML **DOMPurify** kabi library bilan sanitize qilinishi shart:

```tsx
import DOMPurify from 'dompurify';

function UserComment({ html }: { html: string }) {
  const clean = DOMPurify.sanitize(html);
  return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}
```

DOMPurify:
- `<script>` taglar olib tashlaydi
- Event handler'larni filter qiladi
- `javascript:` URL'lar bloklanadi
- Whitelist asosida xavfsiz tag'lar qoldiriladi

### Alternativalar

```tsx
// ❌ Avoid — raw HTML
<div dangerouslySetInnerHTML={{ __html: '<strong>Bold</strong>' }} />

// ✅ Better — JSX
<div><strong>Bold</strong></div>
```

Markdown content uchun **markdown-to-jsx**:

```tsx
import Markdown from 'markdown-to-jsx';

function MarkdownContent({ source }: { source: string }) {
  return <Markdown>{source}</Markdown>;
}
```

### XSS attack vector misol

```tsx
const comment = userInput;  // "<img src=x onerror=alert('XSS')>"

// ❌ Xavfli
<div dangerouslySetInnerHTML={{ __html: comment }} />
// Image load fail → onerror chaqiriladi → alert ishlaydi
// Real attack: cookie steal, session hijack

// ✅ Xavfsiz
<div>{comment}</div>
// Renders escaped text

// ✅ Sanitized
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(comment) }} />
```

> **Eslatma:** XSS — "Cross-Site Scripting". Untrusted input HTML/JS sifatida bajariladi. Web security'ning eng katta vector'laridan biri.

<details>
<summary><strong>Under the Hood</strong></summary>

**React DOM internal:**

```typescript
function setProperty(node, key, value) {
  if (key === 'dangerouslySetInnerHTML') {
    if (value && '__html' in value) {
      node.innerHTML = value.__html;
      // HTML parsed va DOM tree quriladi
    }
  }
}
```

**`<script>` tag `innerHTML`'da:**

```typescript
const div = document.createElement('div');
div.innerHTML = '<script>alert("XSS")</script>';
// Script DOM'ga qo'shiladi, lekin EXECUTE QILINMAYDI
```

Lekin boshqa vector'lar:

```typescript
div.innerHTML = '<img src="x" onerror="alert(1)">';
// Image load fail → onerror trigger → alert
```

**Sanitization library — DOMPurify:**

```typescript
function sanitize(html: string): string {
  // 1. HTML parse to DOM tree
  const doc = new DOMParser().parseFromString(html, 'text/html');
  
  // 2. Walk tree, har element uchun:
  for (const node of doc.body.querySelectorAll('*')) {
    if (!ALLOWED_TAGS.includes(node.tagName)) {
      node.remove();
      continue;
    }
    
    for (const attr of [...node.attributes]) {
      if (attr.name.startsWith('on')) {
        node.removeAttribute(attr.name);
      }
      if (attr.value.startsWith('javascript:')) {
        node.removeAttribute(attr.name);
      }
    }
  }
  
  return doc.body.innerHTML;
}
```

**XSS protection layers:**

1. **HTML escape** — React default (JSX expression)
2. **`dangerouslySetInnerHTML` ko'rsatkichi** — explicit opt-out
3. **Sanitization** — DOMPurify
4. **Content Security Policy (CSP)** — server header
5. **Input validation** — server-side

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Trusted markdown rendering:

```tsx
import { useMemo } from 'react';
import DOMPurify from 'dompurify';
import { marked } from 'marked';

interface MarkdownContentProps {
  source: string;
}

function MarkdownContent({ source }: MarkdownContentProps) {
  const sanitizedHtml = useMemo(() => {
    const rawHtml = marked.parse(source);
    return DOMPurify.sanitize(rawHtml, {
      ALLOWED_TAGS: ['p', 'h1', 'h2', 'h3', 'strong', 'em', 'a', 'ul', 'ol', 'li', 'code', 'pre'],
      ALLOWED_ATTR: ['href', 'class'],
    });
  }, [source]);

  return (
    <div
      className="markdown-content"
      dangerouslySetInnerHTML={{ __html: sanitizedHtml }}
    />
  );
}
```

XSS-safe — JSX preferred:

```tsx
// ❌ Avoid — raw HTML keraksiz
function UserBio({ bio }: { bio: string }) {
  return <div dangerouslySetInnerHTML={{ __html: bio }} />;
}

// ✅ Better — JSX
function UserBio({ bio }: { bio: string }) {
  return <div>{bio}</div>;
}

// ✅ Best — formatted bio
function UserBio({ bio }: { bio: FormattedBio }) {
  return (
    <div>
      <p>{bio.text}</p>
      {bio.links.map((link, i) => (
        <a key={i} href={link.url} target="_blank" rel="noopener noreferrer">
          {link.text}
        </a>
      ))}
    </div>
  );
}
```

Anti-pattern misol:

```tsx
// ❌ NOTO'G'RI — direct user input
function Comment({ comment }: { comment: string }) {
  return <div dangerouslySetInnerHTML={{ __html: comment }} />;
}

// ✅ TO'G'RI — sanitized
import DOMPurify from 'dompurify';

function Comment({ comment }: { comment: string }) {
  const clean = DOMPurify.sanitize(comment);
  return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}

// ✅ ENG TO'G'RI — JSX
function Comment({ comment }: { comment: string }) {
  return <div>{comment}</div>;
}
```

</details>

---

## Conditional Rendering

### Nazariya

React'da conditional rendering pattern'lari:

### 1. Ternary operator

```tsx
function Greeting({ user }: { user: User | null }) {
  return user
    ? <p>Salom, {user.name}!</p>
    : <p>Login qiling</p>;
}

// Ichida:
<div>
  {user ? <p>Welcome</p> : <p>Login</p>}
</div>
```

### 2. Logical AND (`&&`)

```tsx
function Notification({ count }: { count: number }) {
  return (
    <div>
      <h1>Inbox</h1>
      {count > 0 && <Badge count={count} />}
    </div>
  );
}
```

### 3. The 0 Trap (eng ko'p uchraydigan bug)

```tsx
function ItemList({ items }: { items: Item[] }) {
  return (
    <div>
      <h1>Items</h1>
      {items.length && <Badge count={items.length} />}
    </div>
  );
}

// items.length === 0:
// {0 && <Badge />}
// → 0 falsy → React skip qiladi? YO'Q!
// React: 0 — render qilinadigan number → text "0"
// DOM: <div><h1>Items</h1>0</div>  ← bug!
```

**Sabab:** `&&` operator falsy operand'ni qaytaradi (`0 && X` → `0`). React `0` ni text node sifatida render qiladi.

**Yechimlar:**

```tsx
// ✅ 1. Boolean conversion
{Boolean(items.length) && <Badge count={items.length} />}

// ✅ 2. Explicit comparison
{items.length > 0 && <Badge count={items.length} />}

// ✅ 3. Ternary
{items.length ? <Badge count={items.length} /> : null}

// ✅ 4. Double NOT
{!!items.length && <Badge count={items.length} />}
```

> **Eslatma:** ESLint plugin `react/jsx-no-leaked-render` bu xato'ni topadi.

### 4. Early return (guard clauses)

```tsx
function UserProfile({ user }: { user: User | null }) {
  if (!user) return <Login />;
  if (user.isLoading) return <Skeleton />;
  if (user.error) return <ErrorMessage error={user.error} />;
  
  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}
```

### 5. Render map pattern

```tsx
type ViewType = 'admin' | 'user' | 'guest';

const views: Record<ViewType, ReactElement> = {
  admin: <AdminPanel />,
  user: <UserDashboard />,
  guest: <LandingPage />,
};

function App({ viewType }: { viewType: ViewType }) {
  return views[viewType];
}
```

Lazy evaluation kerak bo'lsa — function map. `views` object ichida `<AdminPanel />` har komponent render'da har bir variant uchun `_jsx(AdminPanel, ...)` chaqiriladi (faqat element object yaratiladi — komponent function'i chaqirilmaydi, shu sababli "performance" jihatidan farq deyarli yo'q). Lekin agar inline JSX'da `useState` chaqirig'i yoki side effect bo'lsa (anti-pattern, lekin uchraydi) — lazy factory faqat tanlangan variantni ishlatadi:

```tsx
const viewFactories: Record<ViewType, () => ReactElement> = {
  admin: () => <AdminPanel />,
  user: () => <UserDashboard />,
  guest: () => <LandingPage />,
};

function App({ viewType }: { viewType: ViewType }) {
  return viewFactories[viewType]();
}
```

### 6. `null` qaytarish

```tsx
function Banner({ show, message }: BannerProps) {
  if (!show) return null;
  return <div className="banner">{message}</div>;
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

**0 trap mexanizmi:**

```tsx
{0 && <Component />}
// JS evaluation: 0 && X → 0
// React reconcile: typeof 0 === 'number' → text fiber
// DOM: text node "0"

{false && <Component />}
// JS evaluation: false && X → false
// React reconcile: typeof false === 'boolean' → null (skip)
// DOM: nothing
```

`Boolean(0)` → `false` — explicit conversion bug oldini olish uchun.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

0 trap real misol:

```tsx
// ❌ Production'da uchraydigan bug
function CartIndicator({ cart }: { cart: { items: Item[] } }) {
  return (
    <div>
      Cart {cart.items.length && <Badge count={cart.items.length} />}
    </div>
  );
}

// Cart bo'sh:
// items.length === 0
// {0 && <Badge />} → 0
// DOM: <div>Cart 0</div>  ← "0" ko'rinadi!

// ✅ Yechim 1: comparison
function CartIndicator({ cart }: { cart: { items: Item[] } }) {
  return (
    <div>
      Cart {cart.items.length > 0 && <Badge count={cart.items.length} />}
    </div>
  );
}

// ✅ Yechim 2: ternary
function CartIndicator({ cart }: { cart: { items: Item[] } }) {
  return (
    <div>
      Cart {cart.items.length ? <Badge count={cart.items.length} /> : null}
    </div>
  );
}
```

Multi-state UI:

```tsx
type Status = 'loading' | 'success' | 'error' | 'idle';

interface DataViewProps {
  status: Status;
  data?: Data;
  error?: Error;
}

function DataView({ status, data, error }: DataViewProps) {
  if (status === 'idle') return <p>Click load to fetch data</p>;
  if (status === 'loading') return <Skeleton />;
  if (status === 'error') return <ErrorMessage error={error} />;
  if (status === 'success' && data) return <DataDisplay data={data} />;
  
  return null;
}
```

Render map pattern:

```tsx
type Tab = 'overview' | 'details' | 'reviews' | 'related';

interface ProductPageProps {
  product: Product;
  activeTab: Tab;
}

function ProductPage({ product, activeTab }: ProductPageProps) {
  const tabContent: Record<Tab, () => ReactElement> = {
    overview: () => <Overview product={product} />,
    details: () => <Details product={product} />,
    reviews: () => <Reviews productId={product.id} />,
    related: () => <RelatedProducts category={product.category} />,
  };

  return (
    <div>
      <ProductHeader product={product} />
      <TabBar active={activeTab} />
      <main>{tabContent[activeTab]()}</main>
    </div>
  );
}
```

Conditional className:

```tsx
function Button({ variant, isActive, isDisabled }: ButtonProps) {
  const classes = [
    'btn',
    `btn-${variant}`,
    isActive && 'active',
    isDisabled && 'disabled',
  ]
    .filter(Boolean)
    .join(' ');

  return <button className={classes} disabled={isDisabled}>Click</button>;
}

// Yoki classnames library bilan:
import clsx from 'clsx';

function Button({ variant, isActive, isDisabled }: ButtonProps) {
  return (
    <button
      className={clsx(
        'btn',
        `btn-${variant}`,
        { active: isActive, disabled: isDisabled }
      )}
      disabled={isDisabled}
    >
      Click
    </button>
  );
}
```

</details>

---

## Edge Cases va Gotchas

### Whitespace handling

```tsx
// Bir qator
<div>Hello, world!</div>
// Hosil: "Hello, world!"

// Element orasidagi whitespace
<p>
  <span>A</span> <span>B</span>
</p>
// Hosil: "A B" (1 ta probel)

<p>
  <span>A</span>
  <span>B</span>
</p>
// Hosil: "AB" (newline trim, no probel)

// Manual probel
<p>
  <span>A</span>
  {' '}
  <span>B</span>
</p>
// Hosil: "A B"
```

---

### React Element identity

```tsx
const element = <div>Hello</div>;

console.log(element === <div>Hello</div>);  // false
// Har JSX expression yangi element yaratadi
```

---

### Type narrowing TS bilan

```tsx
function App({ user }: { user: User | null }) {
  // ✅ TS narrowing — early return
  if (!user) return null;
  
  return <p>{user.name}</p>;
}
```

---

### Children prop

```tsx
interface CardProps {
  children: ReactNode;
}

function Card({ children }: CardProps) {
  return <div className="card">{children}</div>;
}

// Ishlatish:
<Card>{42}</Card>
<Card>Hello</Card>
<Card><h1>Title</h1></Card>
<Card><h1>Title</h1><p>Body</p></Card>
<Card>{null}</Card>
<Card>{condition && <X />}</Card>
```

---

## Common Mistakes

### ❌ Xato 1: 0 trap

```tsx
{count && <Badge />}  // count=0 → "0" render qilinadi

// ✅
{count > 0 && <Badge />}
```

---

### ❌ Xato 2: `class` ishlatish

```tsx
<div class="primary">...</div>

// ✅
<div className="primary">...</div>
```

---

### ❌ Xato 3: Self-close yo'q

```tsx
<img src="..." alt="...">  // Syntax error

// ✅
<img src="..." alt="..." />
```

---

### ❌ Xato 4: Event handler chaqirish

```tsx
<button onClick={handleClick()}>Click</button>
// handleClick() darhol chaqiriladi

// ✅
<button onClick={handleClick}>Click</button>

// Argument bilan
<button onClick={() => handleClick(id)}>Click</button>
```

---

### ❌ Xato 5: Object children

```tsx
const user = { name: 'Ali' };
<div>{user}</div>  // Error: Objects are not valid as a React child

// ✅
<div>{user.name}</div>
<div>{JSON.stringify(user)}</div>
```

---

## Amaliy Mashqlar

### Mashq 1: JSX vs HTML conversion (Oson)

Quyidagi HTML'ni JSX'ga o'tkazing:

```html
<form action="/submit" method="POST">
  <label for="name">Name:</label>
  <input type="text" id="name" class="input" maxlength="50" required>
  <input type="checkbox" id="agree" checked>
  <label for="agree">Agree</label>
  <button type="submit" tabindex="0">Submit</button>
</form>
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function MyForm() {
  return (
    <form action="/submit" method="POST">
      <label htmlFor="name">Name:</label>
      <input
        type="text"
        id="name"
        className="input"
        maxLength={50}
        required
      />
      <input
        type="checkbox"
        id="agree"
        checked={true}
        readOnly
      />
      <label htmlFor="agree">Agree</label>
      <button type="submit" tabIndex={0}>Submit</button>
    </form>
  );
}
```

Tuzatishlar:
- `for` → `htmlFor`
- `class` → `className`
- `maxlength` → `maxLength={50}` (number)
- `tabindex` → `tabIndex={0}`
- `<input>` → `<input ... />` (self-close)
- `checked={true}` + `readOnly` — `checked` prop berilgan bo'lsa, React uni controlled input deb hisoblaydi va `onChange` (yoki `readOnly`) talab qiladi. Boshqa variant: `defaultChecked` (uncontrolled) — initial value berib, keyingi state'ni DOM o'zi boshqarsin

</details>

---

### Mashq 2: 0 trap fix (Oson)

Quyidagi kodni tuzating:

```tsx
function Inbox({ messages }: { messages: Message[] }) {
  return (
    <div>
      <h1>Inbox</h1>
      {messages.length && <Badge count={messages.length} />}
      {messages.map(m => <MessageItem key={m.id} message={m} />)}
    </div>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

**Bug:** Agar `messages.length === 0` bo'lsa, `{0 && <Badge />}` → `0` render qilinadi.

**Tuzatish:**

```tsx
function Inbox({ messages }: { messages: Message[] }) {
  return (
    <div>
      <h1>Inbox</h1>
      {messages.length > 0 && <Badge count={messages.length} />}
      {messages.map(m => <MessageItem key={m.id} message={m} />)}
    </div>
  );
}
```

</details>

---

### Mashq 3: Spread attribute (O'rta)

`Input` komponent yozing — barcha native input attribute'larni qabul qiladi va `error` prop bilan styling.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { InputHTMLAttributes, Ref, useId } from 'react';

interface InputProps extends InputHTMLAttributes<HTMLInputElement> {
  error?: string;
  label?: string;
  ref?: Ref<HTMLInputElement>;
}

// R19: ref oddiy prop — forwardRef keraksiz
function Input({ error, label, className, id, ref, ...rest }: InputProps) {
  // React'ning `useId()` hook'i — SSR-safe, deterministic, hydration uchun stable.
  // `Math.random()` render ichida — har render'da yangi ID, hydration mismatch xavfi.
  const reactId = useId();
  const inputId = id ?? reactId;

  return (
    <div className="input-group">
      {label && <label htmlFor={inputId}>{label}</label>}
      <input
        ref={ref}
        id={inputId}
        className={`input ${error ? 'input-error' : ''} ${className ?? ''}`}
        {...rest}
      />
      {error && <span className="error-text">{error}</span>}
    </div>
  );
}

<Input
  type="email"
  label="Email"
  placeholder="example@example.com"
  required
  maxLength={100}
  autoComplete="email"
  error={emailError}
  data-testid="email-input"
/>
```

</details>

---

### Mashq 4: Conditional rendering refactor (O'rta)

```tsx
function StatusBadge({ status }: { status: 'loading' | 'success' | 'error' | 'idle' }) {
  return status === 'loading'
    ? <Spinner />
    : status === 'success'
    ? <CheckIcon />
    : status === 'error'
    ? <ErrorIcon />
    : <IdleIcon />;
}
```

<details>
<summary><strong>Javob</strong></summary>

**Variant 1: Early returns**

```tsx
function StatusBadge({ status }: { status: 'loading' | 'success' | 'error' | 'idle' }) {
  if (status === 'loading') return <Spinner />;
  if (status === 'success') return <CheckIcon />;
  if (status === 'error') return <ErrorIcon />;
  return <IdleIcon />;
}
```

**Variant 2: Render map**

```tsx
function StatusBadge({ status }: { status: 'loading' | 'success' | 'error' | 'idle' }) {
  const icons = {
    loading: <Spinner />,
    success: <CheckIcon />,
    error: <ErrorIcon />,
    idle: <IdleIcon />,
  };
  
  return icons[status];
}
```

**Variant 3: Switch (TS exhaustive check)**

```tsx
function StatusBadge({ status }: { status: 'loading' | 'success' | 'error' | 'idle' }) {
  switch (status) {
    case 'loading': return <Spinner />;
    case 'success': return <CheckIcon />;
    case 'error': return <ErrorIcon />;
    case 'idle': return <IdleIcon />;
    default: {
      const _exhaustive: never = status;
      throw new Error(`Unhandled status: ${_exhaustive}`);
    }
  }
}
```

</details>

---

### Mashq 5: XSS-safe rendering (Qiyin)

Foydalanuvchi inputi'dan keladigan markdown'ni xavfsiz render qiluvchi `SafeMarkdown` komponent yozing.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useMemo } from 'react';
import DOMPurify from 'dompurify';
import { marked } from 'marked';

interface SafeMarkdownProps {
  source: string;
  className?: string;
}

function SafeMarkdown({ source, className }: SafeMarkdownProps) {
  const html = useMemo(() => {
    const rawHtml = marked.parse(source, { async: false }) as string;
    
    return DOMPurify.sanitize(rawHtml, {
      ALLOWED_TAGS: [
        'p', 'br', 'strong', 'em', 'u', 'code', 'pre',
        'h1', 'h2', 'h3', 'h4', 'h5', 'h6',
        'ul', 'ol', 'li',
        'a', 'blockquote',
      ],
      ALLOWED_ATTR: ['href', 'class', 'id'],
      ALLOWED_URI_REGEXP: /^(?:(?:https?|ftp|mailto):|[^a-z]|[a-z+.\-]+(?:[^a-z+.\-:]|$))/i,
    });
  }, [source]);

  return (
    <div
      className={`markdown-content ${className ?? ''}`}
      dangerouslySetInnerHTML={{ __html: html }}
    />
  );
}

function CommentSection() {
  const userComment = "<script>alert('XSS')</script>**Bold text**";
  
  return (
    <div>
      <h2>Comment:</h2>
      <SafeMarkdown source={userComment} />
    </div>
  );
}
```

**Defense layers:**

1. `useMemo` — har source uchun bir marta sanitize
2. `marked` — markdown → HTML aylantirish
3. `DOMPurify.sanitize` — XSS attack vector'larni filter qilish
4. `ALLOWED_TAGS` whitelist
5. `ALLOWED_ATTR` whitelist
6. `ALLOWED_URI_REGEXP` — `javascript:` URL'lar bloklash

XSS — multi-layer defense bilan oldini olish.

</details>

---

## Xulosa

Bu bo'limda JSX'ning barcha asosiy qoidalari yoritildi:

- **JSX vs TSX** — bir xil syntax, runtime'da farq yo'q
- **JSX = JS expression** — `_jsx(type, props)` chaqiruvi
- **JSX vs HTML farqlari** — `className`, `htmlFor`, camelCase, style object, onChange semantikasi
- **Transform** — Classic vs Automatic (R17+)
- **Expressions** — `{}` ichida JS, statement TAQIQ
- **Single root + Fragments** — JSX qoidasi, `<></>` extra DOM yo'q
- **Reserved attributes** — JS keyword + camelCase
- **Self-closing + Boolean** — `<img />`, `disabled={true}`
- **Spread attributes** — `{...props}`, override hazardlari
- **HTML insertion API + XSS** — sanitization majburiy
- **Conditional rendering** — ternary, `&&`, **0 trap**, early returns, render map

JSX — React'ning syntactic foundation. Keyingi bo'limda lists rendering va `key` prop chuqurroq yoritiladi.

---

**Keyingi bo'lim:** [08-list-rendering.md](08-list-rendering.md) — List Rendering: `map()` array → JSX, `key` prop chuqur, key qoidalari, index as key qachon OK/qachon yomon, nested lists, key collisions, reordering performance, Reconciler bilan bog'lanish.
