# Bo'lim 38: Web Components Interop

> Web Components — brauzer-native komponent standarti to'plami: **Custom Elements** (`HTMLElement` extend), **Shadow DOM** (style va DOM isolation), **HTML Templates** (`<template>`/`<slot>` reusable markup), va **Custom Events** (`dispatchEvent` orqali DOM-level signal). Pre-R19'da React Custom Element'lar bilan bir muammo bor edi: complex props (object, function, array) HTML attribute sifatida string'ga konvertatsiya qilinardi va `[object Object]` qilib pass qilinardi. **R19** bu muammoni hal qildi: React endi primitive (string/number/boolean) bo'lganda attribute sifatida, complex bo'lganda DOM property sifatida set qiladi. Bu fayl Custom Elements API, Lifecycle Callbacks, Pre-R19 cheklovlar, R19 yechimi, Shadow DOM va Slots integratsiyasi, Custom Events handling, TypeScript JSX intrinsic elements augmentation, va Web Component vs React Component Decision Guide'ni qamrab oladi.

---

## Mundarija

- [Web Components Ekosistema](#web-components-ekosistema)
- [Custom Elements API](#custom-elements-api)
- [Lifecycle Callbacks](#lifecycle-callbacks)
- [Properties vs Attributes — Pre-R19 Muammo](#properties-vs-attributes--pre-r19-muammo)
- [R19 Properties vs Attributes Yechimi](#r19-properties-vs-attributes-yechimi)
- [React'da Web Component Ishlatish](#reactda-web-component-ishlatish)
- [Custom Events Handling](#custom-events-handling)
- [Shadow DOM](#shadow-dom)
- [Slots — Light DOM va Shadow DOM](#slots--light-dom-va-shadow-dom)
- [React Component Web Component Ichida](#react-component-web-component-ichida)
- [TypeScript JSX Intrinsic Elements](#typescript-jsx-intrinsic-elements)
- [Decision Guide — Web Component vs React Component](#decision-guide--web-component-vs-react-component)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Web Components Ekosistema

### Nazariya

Web Components — brauzer-native API'lar to'plami:

1. **Custom Elements** — yangi HTML element'lar yaratish (`HTMLElement` extend).
2. **Shadow DOM** — element'ning ichki DOM va CSS'ini tashqi document'dan izolyatsiya qilish.
3. **HTML Templates** — `<template>` element reusable markup uchun, `<slot>` Shadow DOM ichida content projection uchun.
4. **HTML Imports** — boshlang'ich Web Components spec'ining qismi edi, lekin standardizatsiyadan chiqarilgan (Chrome'da deprecated). Bu rolni endi ESM (`import` statement) bajaradi — komponent kodi modul sifatida import qilinadi, registratsiya `customElements.define` orqali.

**Maqsad:** Framework-agnostic UI komponent'lar yaratish. Bir marta yozilgan Web Component React, Vue, Svelte, vanilla JS yoki hech qanday framework'siz ishlatilishi mumkin.

**Brauzer support (Custom Elements v1):**

- Chrome 54+, Firefox 63+, Safari 10.1+, Edge 79+ (Chromium-based)
- Shadow DOM v1: Chrome 53+, Firefox 63+, Safari 10+
- Mobile: barcha modern brauzerlar
- IE11: polyfill kerak (deprecated, support yo'q)

**Web Components yozish library'lari:**

- **Lit** (Google) — Web Components yozish uchun reactive primitives (`LitElement`, `html`, `css` template tags), declarative properties via decorators. Lightweight runtime — aniq bundle hajmi versiya va tree-shake'ga bog'liq (`bundlephobia.com/package/lit`).
- **Stencil** (Ionic) — TSX-style syntax, AOT compilation, build-time TypeScript types.
- **Hybrids** — declarative, functional approach.
- **Native** — library'siz, direct `HTMLElement` extend.

**React vs Web Components — fundamental farqlar:**

| Aspect | React | Web Components |
|--------|-------|----------------|
| Runtime | JavaScript framework | Browser-native |
| Style isolation | CSS Modules / CSS-in-JS / Tailwind | Shadow DOM (native, browser-level) |
| Render model | Fiber tree + diff (Virtual DOM diffing) → DOM | Imperative DOM ops (yoki library: Lit lit-html template diff) |
| Communication | Props + callbacks | Attributes + Properties + Events |
| State | useState, hooks | Class fields, signals (Lit `@state`) |
| Bundle | Framework + components | Components only (Lit/Stencil framework runtime qo'shadi, lekin React'dan kichikroq) |
| Cross-framework | React-only ecosystem | Universal |
| Type safety | TS first-class | TS via decorators (Lit/Stencil) |

> **Versiya evolyutsiyasi (Web Components support):**
> - **Pre-R19 (2018-2024):** React Custom Element'larga props'ni HTML attribute sifatida set qilardi. Object/Function/Array → string konvertatsiya → `[object Object]`. Workaround: `useEffect` + `ref.current.propName = value`. Custom event'lar har doim `addEventListener` orqali.
> - **R19 (2024+):** React primitive props attribute sifatida, complex props (object/function/array/non-string) DOM property sifatida set qiladi. Native Custom Element best practices'ni avtomatik qo'llaydi.
> - **Sabab:** Modern Web Component library'lar (Lit) reactive properties API ishlatadi → object/array binding kerak. React-WC interop muammolari developer experience'ni buzgan.

<details>
<summary><strong>Under the Hood</strong></summary>

**Custom Element registration:**

```javascript
class ProductCard extends HTMLElement {
  static get observedAttributes() {
    return ['name', 'price'];
  }

  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
  }

  connectedCallback() {
    this.render();
  }

  attributeChangedCallback(name, oldValue, newValue) {
    if (oldValue !== newValue) this.render();
  }

  set product(value) {
    this._product = value;
    this.render();
  }

  render() {
    this.shadowRoot.innerHTML = `
      <style>
        h3 { color: var(--product-color, #1a1a1a); }
      </style>
      <h3>${this.getAttribute('name') || ''}</h3>
      <p>$${this.getAttribute('price') || '0'}</p>
    `;
  }
}

customElements.define('product-card', ProductCard);
```

**HTML usage:**

```html
<product-card name="iPhone 15" price="999"></product-card>

<script>
  const card = document.querySelector('product-card');
  card.product = { id: 42, name: 'iPhone', specs: {...} }; // ← property assignment
</script>
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Native Custom Element (no library):

```typescript
class HelloElement extends HTMLElement {
  connectedCallback() {
    this.innerHTML = `<p>Hello, ${this.getAttribute('name') || 'World'}!</p>`;
  }
}

customElements.define('hello-element', HelloElement);
```

```html
<hello-element name="Alice"></hello-element>
<!-- Output: <p>Hello, Alice!</p> -->
```

Lit-based Web Component:

```typescript
import { LitElement, html, css } from 'lit';
import { customElement, property } from 'lit/decorators.js';

@customElement('user-greeting')
export class UserGreeting extends LitElement {
  static styles = css`
    :host {
      display: block;
      padding: 16px;
      border: 1px solid #ddd;
      border-radius: 8px;
    }

    h2 {
      color: var(--primary-color, #1a1a1a);
    }
  `;

  @property({ type: String })
  name = 'Guest';

  @property({ type: Object })
  user: { id: number; email: string } | null = null;

  render() {
    return html`
      <h2>Welcome, ${this.name}!</h2>
      ${this.user ? html`<p>Email: ${this.user.email}</p>` : null}
    `;
  }
}
```

```html
<user-greeting name="Alice"></user-greeting>

<script type="module">
  const greeting = document.querySelector('user-greeting');
  greeting.user = { id: 1, email: 'alice@example.com' };
</script>
```

</details>

---

## Custom Elements API

### Nazariya

Custom Elements — DOM'da yangi element turini ro'yxatga olish uchun standart API. Element nomida **kamida bitta defis** bo'lishi shart (`<my-element>`, `<user-card>`, emas `<myelement>`).

**Definition:**

```javascript
customElements.define('element-name', ElementClass, options?);
```

- **`element-name`** — kebab-case, defis majburiy.
- **`ElementClass`** — `HTMLElement` (yoki uning child class) extend qiluvchi class.
- **`options.extends`** — built-in element'ni kengaytirish (`{ extends: 'button' }` for `<button is="my-button">`).

**Element types:**

1. **Autonomous** — `<my-card>`, default `HTMLElement` extend.
2. **Customized built-in** — `<button is="my-button">`, ma'lum built-in element extend. Apple WebKit team bu API'ni hech qachon implement qilmagan (Safari'da `is` attribute qo'llab-quvvatlanmaydi) — bu **deprecated emas, lekin cross-browser uchun mavjud emas**. Tavsiya: faqat autonomous Custom Element'lar ishlatish.

**API methods:**

```javascript
customElements.define('x-card', XCard);          // Register
customElements.get('x-card');                    // Returns class
customElements.whenDefined('x-card');            // Returns Promise (resolved when defined)
customElements.upgrade(element);                // Manual upgrade
```

**Constructor restrictions:**

```javascript
class XCard extends HTMLElement {
  constructor() {
    super();

    // ❌ TAQIQ: attribute getter/setter
    // this.getAttribute('name'); // TypeError

    // ❌ TAQIQ: child element manipulation
    // this.appendChild(...);

    // ❌ TAQIQ: parent navigation
    // this.parentNode;

    // ✅ OK: shadow DOM attach
    this.attachShadow({ mode: 'open' });
  }
}
```

Constructor'da DOM manipulation taqiqlanadi — element hali tree'ga insert qilinmagan. `connectedCallback`'ni ishlatish kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

**Custom Elements registry — internal structure:**

```javascript
// Browser internal (taxminiy):
class CustomElementRegistry {
  #registry = new Map<string, CustomElementDefinition>();
  #pendingUpgrades = new Map<string, Promise<void>>();

  define(name, constructor, options) {
    this.#validateName(name);
    this.#registry.set(name, { constructor, options });

    // Existing elements upgrade
    document.querySelectorAll(name).forEach((element) => {
      this.#upgradeElement(element, constructor);
    });
  }

  #upgradeElement(element, constructor) {
    Object.setPrototypeOf(element, constructor.prototype);
    constructor.call(element);
    if (element.isConnected) {
      element.connectedCallback?.();
    }
  }
}
```

**Registration timing:**

- Custom Element class registratsiyasidan **oldin** HTML'da yozilsa — element "uninitialized" holatda (oddiy `HTMLElement` instance).
- Registratsiyadan **keyin** — element "upgraded" — `connectedCallback` chaqiriladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Native autonomous element:

```typescript
class CounterElement extends HTMLElement {
  private count = 0;

  constructor() {
    super();
    const shadow = this.attachShadow({ mode: 'open' });
    shadow.innerHTML = `
      <button>Click: <span>${this.count}</span></button>
    `;
  }

  connectedCallback() {
    const button = this.shadowRoot!.querySelector('button')!;
    const span = this.shadowRoot!.querySelector('span')!;

    button.addEventListener('click', () => {
      this.count++;
      span.textContent = String(this.count);

      this.dispatchEvent(
        new CustomEvent('count-change', {
          detail: { count: this.count },
          bubbles: true,
          composed: true,
        })
      );
    });
  }
}

customElements.define('counter-element', CounterElement);
```

`whenDefined` async loading:

```typescript
async function ensureCustomElement() {
  await customElements.whenDefined('product-card');
  // Endi <product-card> ishlatilishi mumkin
}

// Yoki Promise.all bilan multiple:
async function setup() {
  await Promise.all([
    customElements.whenDefined('product-card'),
    customElements.whenDefined('user-greeting'),
    customElements.whenDefined('rich-editor'),
  ]);

  // Barcha komponentlar ready
}
```

Custom Element registry check:

```typescript
function isElementDefined(name: string): boolean {
  return customElements.get(name) !== undefined;
}

if (!isElementDefined('product-card')) {
  await import('./components/product-card.js');
  // Module side effect orqali register qilinadi
}
```

</details>

---

## Lifecycle Callbacks

### Nazariya

Custom Elements 4 ta hayot davriy callback'lar:

1. **`connectedCallback()`** — element DOM'ga insert qilinganda.
2. **`disconnectedCallback()`** — element DOM'dan o'chirilganda.
3. **`attributeChangedCallback(name, oldValue, newValue)`** — observed attribute o'zgarganda.
4. **`adoptedCallback()`** — element yangi document'ga ko'chirilganda (kamdan-kam, iframe context).

**`observedAttributes` getter:**

```javascript
class XCard extends HTMLElement {
  static get observedAttributes() {
    return ['name', 'value', 'disabled'];
  }

  attributeChangedCallback(name, oldValue, newValue) {
    // Faqat observedAttributes'dagi attribute'lar uchun chaqiriladi
  }
}
```

**Callback timing:**

```
1. customElements.define('x-card', XCard)
2. <x-card name="alice"></x-card> appears in DOM
3. constructor() runs
4. attributeChangedCallback('name', null, 'alice')  // initial attributes
5. connectedCallback()
6. (later) element.setAttribute('name', 'bob')
7. attributeChangedCallback('name', 'alice', 'bob')
8. (later) element.remove()
9. disconnectedCallback()
```

**Cleanup pattern:**

```javascript
class XCard extends HTMLElement {
  #abortController = new AbortController();

  connectedCallback() {
    document.addEventListener(
      'click',
      this.handleDocumentClick,
      { signal: this.#abortController.signal }
    );

    this.#interval = setInterval(this.tick, 1000);
  }

  disconnectedCallback() {
    this.#abortController.abort();
    clearInterval(this.#interval);
  }
}
```

<details>
<summary><strong>Kod Misollari</strong></summary>

To'liq lifecycle Custom Element:

```typescript
class TimerElement extends HTMLElement {
  static get observedAttributes() {
    return ['interval', 'autoplay'];
  }

  private intervalId: number | null = null;
  private count = 0;

  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
  }

  connectedCallback() {
    this.render();

    if (this.hasAttribute('autoplay')) {
      this.start();
    }
  }

  disconnectedCallback() {
    this.stop();
  }

  attributeChangedCallback(name: string, oldValue: string | null, newValue: string | null) {
    if (oldValue === newValue) return;

    if (name === 'interval' && this.intervalId !== null) {
      this.stop();
      this.start();
    }

    if (name === 'autoplay') {
      newValue !== null ? this.start() : this.stop();
    }
  }

  start() {
    if (this.intervalId !== null) return;

    const interval = parseInt(this.getAttribute('interval') ?? '1000', 10);
    this.intervalId = window.setInterval(() => {
      this.count++;
      this.render();
      this.dispatchEvent(
        new CustomEvent('tick', { detail: { count: this.count }, bubbles: true })
      );
    }, interval);
  }

  stop() {
    if (this.intervalId !== null) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
  }

  render() {
    if (!this.shadowRoot) return;
    this.shadowRoot.innerHTML = `
      <style>
        :host { display: inline-block; padding: 8px; border: 1px solid #ddd; }
      </style>
      <strong>Count:</strong> ${this.count}
    `;
  }
}

customElements.define('timer-element', TimerElement);
```

```html
<timer-element interval="500" autoplay></timer-element>

<script>
  document.addEventListener('tick', (e) => {
    console.log('Count:', e.detail.count);
  });
</script>
```

AbortController cleanup pattern:

```typescript
class WatcherElement extends HTMLElement {
  private abortController: AbortController | null = null;

  connectedCallback() {
    this.abortController = new AbortController();
    const { signal } = this.abortController;

    window.addEventListener('resize', this.handleResize, { signal });
    document.addEventListener('keydown', this.handleKeyDown, { signal });

    const ws = new WebSocket('wss://api.example.com');
    signal.addEventListener('abort', () => ws.close());
  }

  disconnectedCallback() {
    this.abortController?.abort();
    this.abortController = null;
  }

  private handleResize = () => {
    /* ... */
  };

  private handleKeyDown = (e: KeyboardEvent) => {
    /* ... */
  };
}

customElements.define('watcher-element', WatcherElement);
```

</details>

---

## Properties vs Attributes — Pre-R19 Muammo

### Nazariya

HTML element'larida ikki turdagi data'ni saqlash mumkin:

1. **HTML Attributes** — markup'da yozilgan (`<button disabled>`), faqat **string** turida.
2. **DOM Properties** — JavaScript'da set qilinadi (`button.disabled = true`), **har qanday tur** (object, function, array).

**Pre-R19 React behavior:**

React har JSX prop'ni HTML attribute sifatida set qilardi:

```tsx
<my-element data={complexObject} />

// React:
const element = document.createElement('my-element');
element.setAttribute('data', String(complexObject)); // ← "[object Object]"
```

Bu pattern primitive (string, number) uchun ishlardi, lekin object, function, array uchun fojiali edi:

```tsx
<my-card user={{ id: 42, name: 'Alice' }} />

// HTML output:
// <my-card user="[object Object]"></my-card>  ← FAIL
```

**Pre-R19 workaround — `useEffect` + `ref`:**

```tsx
import { useRef, useEffect } from 'react';

function MyCardWrapper({ user }: { user: User }) {
  const ref = useRef<HTMLElement>(null);

  useEffect(() => {
    if (ref.current) {
      // Imperative property assignment
      (ref.current as any).user = user;
    }
  }, [user]);

  return <my-card ref={ref} />;
}
```

Bu workaround quyidagi problemalarga olib keladi:

- Boilerplate har komponent uchun.
- TypeScript type lost — `as any`.
- React state synchronization murakkab.
- SSR'da effect ishlamaydi.

**Lit'ning aralash strategiyasi:**

Lit `@property` decorator ham attribute, ham property qabul qiladi:

```typescript
@property({ type: Object })
user: User | null = null;
// Sets both attribute (JSON string) and property
```

Lekin React'dan attribute sifatida pass qilinsa string parse qilish kerak — overhead.

<details>
<summary><strong>Kod Misollari</strong></summary>

Pre-R19 manual workaround pattern:

```tsx
import { useRef, useEffect, ReactNode } from 'react';

function CustomElementWrapper<P extends Record<string, unknown>>({
  tagName,
  properties,
  children,
}: {
  tagName: string;
  properties: P;
  children?: ReactNode;
}) {
  const ref = useRef<HTMLElement>(null);

  useEffect(() => {
    if (!ref.current) return;

    for (const [key, value] of Object.entries(properties)) {
      if (typeof value === 'object' || typeof value === 'function') {
        (ref.current as any)[key] = value;
      } else {
        if (value === false || value == null) {
          ref.current.removeAttribute(key);
        } else {
          ref.current.setAttribute(key, String(value));
        }
      }
    }
  }, [properties]);

  return React.createElement(tagName, { ref }, children);
}

// Ishlatish:
<CustomElementWrapper
  tagName="my-card"
  properties={{
    user: { id: 42, name: 'Alice' },
    onSelect: (id: number) => console.log(id),
    title: 'User Profile',
  }}
/>
```

Object → JSON workaround (uglier):

```tsx
function ProductDisplay({ product }: { product: Product }) {
  return (
    <my-card
      product-json={JSON.stringify(product)}
    />
  );
}

// Web Component side:
class MyCard extends HTMLElement {
  static get observedAttributes() {
    return ['product-json'];
  }

  attributeChangedCallback(name: string, oldValue: string, newValue: string) {
    if (name === 'product-json') {
      try {
        this.product = JSON.parse(newValue); // ← parsing overhead
      } catch (e) {
        console.error('Invalid product-json:', e);
      }
    }
  }
}
```

Bu pattern verbose va error-prone. R19 buni hal qiladi.

</details>

---

## R19 Properties vs Attributes Yechimi

### Nazariya

R19 React JSX prop'larni Custom Element'larga set qilish strategiyasini o'zgartirdi:

**Detection logic (taxminiy — rasmiy React 19 docs asosida):**

```javascript
// React internal (taxminiy soddalashtirilgan):
function setCustomElementProp(element, propName, value) {
  // Object/function/array — har doim DOM property
  if (typeof value === 'object' || typeof value === 'function') {
    element[propName] = value;
    return;
  }

  // Primitives (string, number) — setAttribute
  if (typeof value === 'string' || typeof value === 'number') {
    element.setAttribute(propName, String(value));
    return;
  }

  // boolean true — setAttribute(key, '') (presence)
  if (value === true) {
    element.setAttribute(propName, '');
    return;
  }

  // false / null / undefined — removeAttribute
  if (value === false || value == null) {
    element.removeAttribute(propName);
    return;
  }
}
```

**Decision matrix:**

| Value type | Action |
|------------|--------|
| `string` | `setAttribute` |
| `number` | `setAttribute` (string konvertatsiya) |
| `true` | `setAttribute(key, '')` (presence) |
| `false` / `null` / `undefined` | `removeAttribute` |
| `object` | `element[propName] = value` (property) |
| `function` | `element[propName] = value` (property) |
| `array` | `element[propName] = value` (property — array ham object) |

**Special case — `propName in element`:**

Agar Custom Element class'ida `name` property mavjud bo'lsa va qiymat **string yoki number EMAS** (object, function, boolean) — React property assignment'ga o'tadi:

```typescript
class MyCard extends HTMLElement {
  config: { theme: string } = { theme: 'light' };
  active: boolean = false;
}

// Non-string + propExists → property
<my-card config={{ theme: 'dark' }} />
// React: 'config' in element && typeof value === 'object' → element.config = { theme: 'dark' }

<my-card active={true} />
// React: 'active' in element && typeof value !== 'string' → element.active = true

// String qiymat HAR DOIM attribute (propExists bo'lsa ham):
<my-card name="Alice" />
// React: typeof value === 'string' → element.setAttribute('name', 'Alice')
// (Custom Element o'zi attributeChangedCallback orqali this.name'ga sync qiladi)
```

Bu Lit-style reactive properties uchun ideal — Lit `@property` decorator attribute → property sync'ni avtomatik bajaradi.

**Camelcase preserved:**

R19 Custom Element prop'larida camelCase'ni `kebab-case`'ga aylantirmaydi:

```tsx
<my-card userId={42} firstName="Alice" />

// Pre-R19: <my-card userid="42" firstname="Alice"> (lowercase)
// R19: element.userId = 42; element.firstName = 'Alice'; (preserved)
```

Lekin built-in HTML element'lar uchun React kebab-case konvertatsiya qiladi (`className` → `class`, `htmlFor` → `for`).

<details>
<summary><strong>Under the Hood</strong></summary>

**R19 dispatcher logic (taxminiy):**

```javascript
function setProp(domElement, key, value, props) {
  if (isCustomElement(domElement.tagName)) {
    setPropForCustomElement(domElement, key, value);
  } else {
    setPropForBuiltinElement(domElement, key, value);
  }
}

function setPropForCustomElement(element, key, value) {
  if (key === 'children' || key === 'ref' || key === 'key') return;
  if (key === 'className') {
    if (value != null) element.className = String(value);
    return;
  }
  if (key === 'style') {
    setStyleProp(element, value);
    return;
  }

  const isObjectType = typeof value === 'object' || typeof value === 'function';
  const propExists = key in element;

  if (isObjectType || (propExists && typeof value !== 'string')) {
    element[key] = value;
  } else if (typeof value === 'string' || typeof value === 'number') {
    element.setAttribute(key, String(value));
  } else if (value === true) {
    element.setAttribute(key, '');
  } else {
    element.removeAttribute(key);
  }
}
```

**Performance considerations:**

- `propName in element` — O(1) lookup (prototype chain traversal).
- Property assignment — O(1) field set + setter trigger.
- Attribute assignment — O(1) string operation, lekin `attributeChangedCallback` trigger.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

R19 native Web Component usage:

```tsx
function ProductPage({ product, currentUser }: { product: Product; currentUser: User }) {
  return (
    <product-card
      productId={product.id}
      productName={product.name}
      product={product}
      onPurchase={() => buyProduct(product.id)}
      currentUser={currentUser}
      featured={true}
    />
  );
}

// React behavior:
// productId={product.id}            → number → setAttribute("productId", "42")
// productName={product.name}        → string → setAttribute("productName", "iPhone 15")
// product={product}                 → object → element.product = product (DOM property)
// onPurchase={() => buyProduct(...)} → function → element.onPurchase = fn (DOM property)
// currentUser={currentUser}         → object → element.currentUser = user
// featured={true}                   → boolean true → setAttribute("featured", "")
//                                      (Lit `@property({ type: Boolean, reflect: true })` kabi
//                                       reactive properties attribute'dan o'qiladi)
```

Lit Web Component (proper consume):

```typescript
import { LitElement, html } from 'lit';
import { customElement, property } from 'lit/decorators.js';

interface Product {
  id: number;
  name: string;
  price: number;
}

interface User {
  id: number;
  email: string;
}

@customElement('product-card')
export class ProductCard extends LitElement {
  @property({ type: Number })
  productId = 0;

  @property({ type: String })
  productName = '';

  @property({ type: Object })
  product: Product | null = null;

  @property({ type: Object })
  currentUser: User | null = null;

  @property({ type: Boolean })
  featured = false;

  @property({ attribute: false })
  onPurchase: (() => void) | null = null;

  render() {
    if (!this.product) return html`<p>Loading...</p>`;

    return html`
      <div class=${this.featured ? 'featured' : ''}>
        <h3>${this.product.name}</h3>
        <p>$${this.product.price}</p>
        <button @click=${this.onPurchase}>Sotib olish</button>
      </div>
    `;
  }
}
```

Conditional rendering with R19:

```tsx
function CartItem({ item, onRemove }: { item: CartItem; onRemove: () => void }) {
  return (
    <cart-item
      itemId={item.id}
      product={item.product}
      quantity={item.quantity}
      onRemove={onRemove}
      removable={!item.locked}
    />
  );
}
```

</details>

---

## React'da Web Component Ishlatish

### Nazariya

R19 Web Component'larni React JSX'da oddiy element sifatida ishlatish imkonini beradi. 3 ta asosiy use case:

1. **Library import** — Material Web Components, Shoelace, Lit Element library'lar.
2. **In-house Web Components** — komandaning umumiy library'si (cross-framework).
3. **Legacy migration** — React'ga ko'chmagan Web Component'lar.

**Workflow:**

```tsx
// 1. Web Component import (side effect — register'ladi)
import 'shoelace/dist/components/button/button';

// 2. JSX'da ishlatish
function App() {
  return <sl-button variant="primary">Click me</sl-button>;
}
```

**Ref via `useRef`:**

```tsx
import { useRef, useEffect } from 'react';

function MyComponent() {
  const buttonRef = useRef<HTMLElement>(null);

  useEffect(() => {
    if (buttonRef.current) {
      (buttonRef.current as any).focus();
    }
  }, []);

  return <sl-button ref={buttonRef}>Submit</sl-button>;
}
```

**Children pass qilish:**

Web Component'larda `children` Light DOM'ga inject qilinadi (Shadow DOM ichida `<slot>` orqali ko'rinadi):

```tsx
<sl-card>
  <h2 slot="header">Card Title</h2>
  <p>Card content</p>
  <div slot="footer">Footer actions</div>
</sl-card>
```

`slot` attribute Web Component'ning Shadow DOM'idagi `<slot name="...">` joylariga ulanadi.

<details>
<summary><strong>Kod Misollari</strong></summary>

Material Web Components ishlatish:

```tsx
import '@material/web/button/filled-button';
import '@material/web/textfield/outlined-text-field';
import '@material/web/checkbox/checkbox';

interface FormData {
  email: string;
  rememberMe: boolean;
}

function LoginForm({ onSubmit }: { onSubmit: (data: FormData) => void }) {
  const [email, setEmail] = useState('');
  const [rememberMe, setRememberMe] = useState(false);

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault();
        onSubmit({ email, rememberMe });
      }}
    >
      <md-outlined-text-field
        label="Email"
        type="email"
        value={email}
        onInput={(e: any) => setEmail(e.target.value)}
        required
      />

      <md-checkbox
        checked={rememberMe}
        onChange={(e: any) => setRememberMe(e.target.checked)}
      >
        Remember me
      </md-checkbox>

      <md-filled-button type="submit">Login</md-filled-button>
    </form>
  );
}
```

Shoelace components:

```tsx
import '@shoelace-style/shoelace/dist/components/button/button';
import '@shoelace-style/shoelace/dist/components/dialog/dialog';
import '@shoelace-style/shoelace/dist/components/input/input';

function ConfirmDialog({
  open,
  title,
  message,
  onConfirm,
  onClose,
}: {
  open: boolean;
  title: string;
  message: string;
  onConfirm: () => void;
  onClose: () => void;
}) {
  return (
    <sl-dialog open={open} label={title}>
      <p>{message}</p>

      <sl-button slot="footer" variant="default" onClick={onClose}>
        Bekor qilish
      </sl-button>
      <sl-button slot="footer" variant="primary" onClick={onConfirm}>
        Tasdiqlash
      </sl-button>
    </sl-dialog>
  );
}
```

Lazy Web Component loading:

```tsx
import { useEffect, useState } from 'react';

function useCustomElement(elementName: string, importFn: () => Promise<unknown>) {
  const [defined, setDefined] = useState(
    () => customElements.get(elementName) !== undefined
  );

  useEffect(() => {
    if (defined) return;

    importFn().then(() => {
      customElements.whenDefined(elementName).then(() => setDefined(true));
    });
  }, [elementName, importFn, defined]);

  return defined;
}

function RichEditorPanel({ content }: { content: string }) {
  const isReady = useCustomElement(
    'rich-editor',
    () => import('./components/rich-editor')
  );

  if (!isReady) return <EditorSkeleton />;

  return <rich-editor initialContent={content} />;
}
```

</details>

---

## Custom Events Handling

### Nazariya

Custom Element'lar `dispatchEvent` orqali DOM-level event'lar emit qiladi:

```javascript
this.dispatchEvent(
  new CustomEvent('item-select', {
    detail: { itemId: 42, name: 'Product' },
    bubbles: true,
    composed: true,
  })
);
```

**React `on*` props limitations:**

React'ning event delegation tizimi faqat ma'lum DOM event'lar uchun ishlaydi (`onClick`, `onChange`, `onInput`, `onKeyDown`, va h.k.). **Custom event'lar uchun ishlamaydi**:

```tsx
// ❌ NOTO'G'RI — React onItemSelect Custom Event'ni eshitmaydi
<my-list onItemSelect={handleSelect} />

// ✅ TO'G'RI — useEffect + addEventListener
const ref = useRef<HTMLElement>(null);

useEffect(() => {
  const handler = (e: Event) => {
    const customEvent = e as CustomEvent<{ itemId: number; name: string }>;
    handleSelect(customEvent.detail.itemId);
  };

  ref.current?.addEventListener('item-select', handler);
  return () => ref.current?.removeEventListener('item-select', handler);
}, [handleSelect]);

return <my-list ref={ref} />;
```

**Pattern wrapping for ergonomic API:**

```tsx
function MyList({ items, onItemSelect }: {
  items: Item[];
  onItemSelect: (id: number) => void;
}) {
  const ref = useRef<HTMLElement>(null);

  useEffect(() => {
    const element = ref.current;
    if (!element) return;

    const handler = (e: Event) => {
      onItemSelect((e as CustomEvent).detail.itemId);
    };

    element.addEventListener('item-select', handler);
    return () => element.removeEventListener('item-select', handler);
  }, [onItemSelect]);

  return <my-list ref={ref} items={items} />;
}
```

**`composed: true` Shadow DOM boundary:**

Default'da Custom Event'lar Shadow DOM boundary'ni **kesib o'tmaydi**. `composed: true` bilan event Light DOM'ga "qayta tug'iladi" va React app uni eshita oladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Event propagation trajectory:**

```
Shadow DOM event:
  shadow-element → ... → shadow-host
                         ↓ (composed: false ← STOP)
  shadow-host → parent → ... → document

Composed event:
  shadow-element → ... → shadow-host
                         ↓ (composed: true)
  shadow-host → parent → ... → document → React handler
```

**`event.composedPath()`:**

```javascript
element.addEventListener('item-select', (e) => {
  console.log(e.composedPath());
  // [shadowChild, shadowHost, ..., body, html, document, window]
});
```

**Performance — `useEffect` cleanup:**

Har render'da `addEventListener` + `removeEventListener` cycle bo'ladi. Optimal:

- `onItemSelect` callback'ni stable reference qilish (`useCallback`).
- `useEffect` deps to'g'ri.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Reusable hook for Custom Event:

```tsx
import { useCallback, useEffect, useRef } from 'react';

function useCustomEvent<T = unknown>(
  eventName: string,
  handler: (detail: T) => void
) {
  const ref = useRef<HTMLElement>(null);
  const handlerRef = useRef(handler);

  handlerRef.current = handler;

  useEffect(() => {
    const element = ref.current;
    if (!element) return;

    const wrappedHandler = (e: Event) => {
      const customEvent = e as CustomEvent<T>;
      handlerRef.current(customEvent.detail);
    };

    element.addEventListener(eventName, wrappedHandler);
    return () => element.removeEventListener(eventName, wrappedHandler);
  }, [eventName]);

  return ref;
}

function ProductList({ products, onSelect }: {
  products: Product[];
  onSelect: (productId: number) => void;
}) {
  const ref = useCustomEvent<{ productId: number }>(
    'product-select',
    ({ productId }) => onSelect(productId)
  );

  return <product-grid ref={ref} products={products} />;
}
```

Multiple events with single hook:

```tsx
function useCustomEvents<T extends Record<string, unknown>>(
  handlers: { [K in keyof T]?: (detail: T[K]) => void }
) {
  const ref = useRef<HTMLElement>(null);
  const handlersRef = useRef(handlers);
  handlersRef.current = handlers;

  // Event nomlari ro'yxatini stable qilish uchun memo'lash kerak — aks holda
  // har render'da yangi keys array → useEffect re-run.
  // Praktikada keys odatda statik (hook fixed event set bilan ishlatiladi),
  // shu bois bo'sh deps OK — lekin agar dynamic key kerak bo'lsa useMemo qo'shing.
  const eventKeys = Object.keys(handlersRef.current);

  useEffect(() => {
    const element = ref.current;
    if (!element) return;

    const subscriptions = eventKeys.map((eventName) => {
      const wrapper = (e: Event) => {
        const customEvent = e as CustomEvent;
        const handler = handlersRef.current[eventName as keyof T];
        handler?.(customEvent.detail);
      };

      element.addEventListener(eventName, wrapper);
      return () => element.removeEventListener(eventName, wrapper);
    });

    return () => subscriptions.forEach((unsub) => unsub());
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [eventKeys.join('|')]);

  return ref;
}

interface ChartEvents {
  'data-point-click': { x: number; y: number; label: string };
  'zoom-change': { scale: number };
  'export-request': { format: 'png' | 'svg' };
}

function InteractiveChart({ data }: { data: ChartData }) {
  const ref = useCustomEvents<ChartEvents>({
    'data-point-click': ({ x, y, label }) => {
      console.log(`Clicked: ${label} at (${x}, ${y})`);
    },
    'zoom-change': ({ scale }) => {
      console.log(`Zoom: ${scale}x`);
    },
    'export-request': ({ format }) => {
      handleExport(format);
    },
  });

  return <chart-component ref={ref} data={data} />;
}
```

Combined with R19 properties:

```tsx
function VideoPlayer({ src, onPlay, onPause, onTimeUpdate }: {
  src: string;
  onPlay: () => void;
  onPause: () => void;
  onTimeUpdate: (time: number) => void;
}) {
  const ref = useCustomEvents({
    'play': () => onPlay(),
    'pause': () => onPause(),
    'time-update': ({ time }: { time: number }) => onTimeUpdate(time),
  });

  return (
    <video-player
      ref={ref}
      src={src}
      autoplay={false}
    />
  );
}
```

</details>

---

## Shadow DOM

### Nazariya

Shadow DOM — Custom Element ichidagi DOM tree'ni asosiy document'dan **izolyatsiya qilish** mexanizmi:

- **CSS isolation** — tashqi style'lar ichkariga ta'sir qilmaydi (va aksincha).
- **DOM isolation** — `document.querySelector` Shadow root ichini ko'rmaydi.
- **Encapsulation** — komponent ichki strukturasi boshqa komponentlarga ta'sir qilmaydi.

**Mode:**

```javascript
this.attachShadow({ mode: 'open' });
this.attachShadow({ mode: 'closed' });
```

**Open vs Closed:**

- **Open** (afzal) — `element.shadowRoot` orqali tashqaridan kirish mumkin.
- **Closed** — `element.shadowRoot === null` (private). Lekin internal ref saqlash kerak (bu pattern ko'pincha unnecessary).

**CSS scoping:**

```css
/* Shadow DOM ichida: */
:host { display: block; padding: 16px; }
:host(.featured) { background: gold; }
:host-context(body.dark) { color: white; }
::slotted(h2) { color: blue; }
```

**Pseudo-classes:**

- `:host` — Custom Element'ning o'zini hostni stylizatsiya qiladi.
- `:host(...)` — host'ning attribute/class bo'yicha conditional.
- `:host-context(...)` — parent context'ga qarab.
- `::slotted(...)` — Light DOM'dan kelgan slotted elementlarni stylizatsiya qiladi.

**CSS variables o'tadi:**

```css
/* Tashqi document: */
my-card { --primary-color: #1a1a1a; }

/* Shadow DOM ichida: */
:host { color: var(--primary-color, black); }
```

CSS variables Shadow DOM boundary'ni kesib o'tadi — komponent author theme variable'larni accept qilishi mumkin.

<details>
<summary><strong>Kod Misollari</strong></summary>

Native Shadow DOM with CSS isolation:

```typescript
class StyledCard extends HTMLElement {
  constructor() {
    super();

    const shadow = this.attachShadow({ mode: 'open' });
    shadow.innerHTML = `
      <style>
        :host {
          display: block;
          padding: 16px;
          border: 1px solid var(--card-border, #ddd);
          border-radius: 8px;
          background: var(--card-bg, white);
        }

        :host(.featured) {
          background: var(--card-featured-bg, gold);
        }

        h2 {
          color: var(--primary-color, #1a1a1a);
          margin: 0 0 8px;
        }

        p {
          color: #666;
          line-height: 1.5;
        }
      </style>

      <h2><slot name="title">Default Title</slot></h2>
      <div><slot></slot></div>
    `;
  }
}

customElements.define('styled-card', StyledCard);
```

```tsx
function App() {
  return (
    <styled-card class="featured" style={{ '--primary-color': '#0066cc' } as any}>
      <h2 slot="title">Featured Article</h2>
      <p>This card has shadow DOM with CSS isolation.</p>
    </styled-card>
  );
}
```

Lit element with constructible stylesheets:

```typescript
import { LitElement, html, css } from 'lit';
import { customElement } from 'lit/decorators.js';

@customElement('themed-button')
export class ThemedButton extends LitElement {
  static styles = css`
    :host {
      display: inline-block;
    }

    button {
      padding: 8px 16px;
      border: none;
      border-radius: 4px;
      background: var(--button-bg, #1a1a1a);
      color: var(--button-color, white);
      font-size: 14px;
      cursor: pointer;
      transition: background 0.2s;
    }

    button:hover {
      background: var(--button-hover-bg, #333);
    }

    button:disabled {
      opacity: 0.5;
      cursor: not-allowed;
    }
  `;

  render() {
    return html`<button><slot></slot></button>`;
  }
}
```

CSS variables in React app:

```tsx
function ThemedApp() {
  return (
    <div
      style={{
        '--button-bg': '#0066cc',
        '--button-hover-bg': '#0052a3',
        '--button-color': 'white',
      } as React.CSSProperties}
    >
      <themed-button>Save</themed-button>
      <themed-button>Cancel</themed-button>
    </div>
  );
}
```

</details>

---

## Slots — Light DOM va Shadow DOM

### Nazariya

`<slot>` elementi Custom Element'ning Shadow DOM'ida "joy" ochadi va Light DOM'dagi children'ni shu joyga "loy" qiladi. Bu pattern composition uchun foydali — komponent struktura'ni boshqaradi, foydalanuvchi content'ni.

**Default slot:**

```html
<!-- Shadow DOM: -->
<div class="card">
  <slot></slot>
</div>

<!-- Light DOM: -->
<my-card>
  <p>This goes into default slot</p>
</my-card>
```

**Named slots:**

```html
<!-- Shadow DOM: -->
<div class="card">
  <header>
    <slot name="title"></slot>
  </header>
  <main>
    <slot></slot>
  </main>
  <footer>
    <slot name="footer"></slot>
  </footer>
</div>

<!-- Light DOM: -->
<my-card>
  <h2 slot="title">Article Title</h2>
  <p>Article body text</p>
  <button slot="footer">Read more</button>
</my-card>
```

**Fallback content:**

```html
<slot name="title">
  <h2>Default Title</h2>
</slot>
```

**`::slotted()` styling:**

```css
:host {
  display: block;
}

::slotted(h2) {
  font-size: 20px;
  color: var(--title-color);
}

::slotted(p) {
  color: #666;
}
```

`::slotted` faqat **direct children** stylizatsiya qiladi, nested element'larga qo'llanmaydi.

<details>
<summary><strong>Kod Misollari</strong></summary>

Tabs Custom Element with named slots:

```typescript
class TabsContainer extends HTMLElement {
  constructor() {
    super();

    const shadow = this.attachShadow({ mode: 'open' });
    shadow.innerHTML = `
      <style>
        :host { display: block; }

        nav {
          display: flex;
          border-bottom: 1px solid #ddd;
        }

        ::slotted([slot="tab"]) {
          padding: 12px 24px;
          cursor: pointer;
          border-bottom: 2px solid transparent;
        }

        ::slotted([slot="tab"][active]) {
          border-bottom-color: #0066cc;
          color: #0066cc;
        }

        main { padding: 16px; }
      </style>

      <nav>
        <slot name="tab"></slot>
      </nav>
      <main>
        <slot name="panel"></slot>
      </main>
    `;
  }
}

customElements.define('tabs-container', TabsContainer);
```

```tsx
function TabsExample() {
  const [activeTab, setActiveTab] = useState('overview');

  return (
    <tabs-container>
      {/* Built-in <button> noma'lum attribute: boolean true → setAttribute("active", ""), false → removeAttribute (React umumiy unknown-attribute logic, R19'ga xos emas). `::slotted([slot="tab"][active])` selector ushbu attribute presence'iga reaksiya beradi. */}
      <button slot="tab" active={activeTab === 'overview'}
              onClick={() => setActiveTab('overview')}>
        Overview
      </button>
      <button slot="tab" active={activeTab === 'specs'}
              onClick={() => setActiveTab('specs')}>
        Specs
      </button>
      <button slot="tab" active={activeTab === 'reviews'}
              onClick={() => setActiveTab('reviews')}>
        Reviews
      </button>

      {activeTab === 'overview' && <div slot="panel">Overview content</div>}
      {activeTab === 'specs' && <div slot="panel">Specs content</div>}
      {activeTab === 'reviews' && <div slot="panel">Reviews content</div>}
    </tabs-container>
  );
}
```

Card layout component:

```typescript
import { LitElement, html, css } from 'lit';
import { customElement } from 'lit/decorators.js';

@customElement('app-card')
export class AppCard extends LitElement {
  static styles = css`
    :host {
      display: block;
      border: 1px solid #e0e0e0;
      border-radius: 8px;
      overflow: hidden;
    }

    header {
      padding: 16px;
      border-bottom: 1px solid #e0e0e0;
      background: #f9f9f9;
    }

    .body {
      padding: 16px;
    }

    footer {
      padding: 16px;
      border-top: 1px solid #e0e0e0;
      background: #f9f9f9;
      display: flex;
      gap: 8px;
      justify-content: flex-end;
    }

    ::slotted(h2) {
      margin: 0;
      font-size: 18px;
    }

    ::slotted(button) {
      padding: 8px 16px;
    }
  `;

  render() {
    return html`
      <header>
        <slot name="header">
          <h2>Untitled</h2>
        </slot>
      </header>
      <div class="body">
        <slot></slot>
      </div>
      <footer>
        <slot name="footer"></slot>
      </footer>
    `;
  }
}
```

```tsx
function ProductDetailsCard({ product, onBuy, onCancel }: {
  product: Product;
  onBuy: () => void;
  onCancel: () => void;
}) {
  return (
    <app-card>
      <h2 slot="header">{product.name}</h2>

      <p>{product.description}</p>
      <strong>${product.price}</strong>

      <button slot="footer" onClick={onCancel}>Cancel</button>
      <button slot="footer" onClick={onBuy}>Buy Now</button>
    </app-card>
  );
}
```

</details>

---

## React Component Web Component Ichida

### Nazariya

Reverse pattern — Web Component ichida React komponent render qilish. Use case: existing Web Component ekosistemasi ichida React-specific feature'larni kiritish (chart library, complex form, AI assistant widget).

**Workflow:**

1. **Custom Element definition** — `connectedCallback`'da React root mount.
2. **Props pass** — Custom Element attributes/properties → React props.
3. **Cleanup** — `disconnectedCallback`'da unmount.
4. **State sync** — bidirectional binding kerak bo'lsa.

**Issues:**

- **Bundle size** — React + komponent har Web Component instance uchun bundle'da bo'lishi kerak.
- **Multiple React versions** — host page React 17, Web Component React 19 bo'lsa konflikt.
- **Shadow DOM events** — React event delegation Shadow DOM boundary bilan farqli ishlaydi.

**Best practices:**

- Web Component'ni minimal qiling — faqat boundary, React tree alohida.
- Single React root per Custom Element instance.
- Cleanup `disconnectedCallback`'da to'liq.

<details>
<summary><strong>Kod Misollari</strong></summary>

React inside Custom Element:

```typescript
import { createRoot, type Root } from 'react-dom/client';
import { Chart } from './Chart';

class ChartWebComponent extends HTMLElement {
  static get observedAttributes() {
    return ['data-source'];
  }

  private root: Root | null = null;
  private container: HTMLDivElement | null = null;

  connectedCallback() {
    if (!this.container) {
      this.container = document.createElement('div');
      this.appendChild(this.container);
      this.root = createRoot(this.container);
    }

    this.render();
  }

  disconnectedCallback() {
    this.root?.unmount();
    this.root = null;
    this.container?.remove();
    this.container = null;
  }

  attributeChangedCallback() {
    this.render();
  }

  private get chartData() {
    const source = this.getAttribute('data-source');
    return source ? JSON.parse(source) : null;
  }

  private render() {
    if (!this.root) return;

    const data = (this as any)._data ?? this.chartData;
    this.root.render(<Chart data={data} />);
  }

  set data(value: ChartData) {
    (this as any)._data = value;
    this.render();
  }

  get data(): ChartData | null {
    return (this as any)._data ?? null;
  }
}

customElements.define('react-chart', ChartWebComponent);
```

```html
<!-- HTML usage (any framework or none): -->
<react-chart></react-chart>

<script>
  const chart = document.querySelector('react-chart');
  chart.data = { values: [1, 2, 3, 4, 5] };
</script>
```

Vue/Angular/vanilla project'da React komponentni kiritish:

```html
<!-- Vue template: -->
<template>
  <react-chart :data-source="JSON.stringify(chartData)"></react-chart>
</template>

<script setup>
import { ref } from 'vue';
import './react-chart-element';

const chartData = ref({ values: [10, 20, 30] });
</script>
```

Shadow DOM bilan React mount:

```typescript
class IsolatedReactRoot extends HTMLElement {
  private root: Root | null = null;

  connectedCallback() {
    const shadow = this.attachShadow({ mode: 'open' });

    const style = document.createElement('style');
    style.textContent = `
      :host { display: block; }
      .container { padding: 16px; }
    `;
    shadow.appendChild(style);

    const container = document.createElement('div');
    container.className = 'container';
    shadow.appendChild(container);

    this.root = createRoot(container);
    this.root.render(<App />);
  }

  disconnectedCallback() {
    this.root?.unmount();
  }
}

customElements.define('isolated-react', IsolatedReactRoot);
```

</details>

---

## TypeScript JSX Intrinsic Elements

### Nazariya

Custom Element'larni JSX'da TypeScript bilan ishlatish uchun `IntrinsicElements` interface'ini kengaytirish kerak. Aks holda TS error: "Property 'my-element' does not exist on type 'JSX.IntrinsicElements'".

**Module augmentation pattern:**

```typescript
// types/custom-elements.d.ts
import type { DetailedHTMLProps, HTMLAttributes } from 'react';

declare module 'react' {
  namespace JSX {
    interface IntrinsicElements {
      'my-card': DetailedHTMLProps<MyCardAttributes, MyCardElement>;
      'my-button': DetailedHTMLProps<MyButtonAttributes, MyButtonElement>;
    }
  }
}

interface MyCardAttributes extends HTMLAttributes<MyCardElement> {
  productId?: number;
  productName?: string;
  product?: Product;
  featured?: boolean;
}

interface MyCardElement extends HTMLElement {
  productId: number;
  productName: string;
  product: Product | null;
  featured: boolean;
}

interface MyButtonAttributes extends HTMLAttributes<MyButtonElement> {
  variant?: 'primary' | 'secondary' | 'danger';
  size?: 'small' | 'medium' | 'large';
  disabled?: boolean;
}

interface MyButtonElement extends HTMLElement {
  variant: 'primary' | 'secondary' | 'danger';
  size: 'small' | 'medium' | 'large';
  disabled: boolean;
}
```

**Lit-style decorator-based types:**

Lit `@property` decorator metadata yaratadi, lekin TS JSX integration alohida step:

```typescript
// component.ts (Lit)
@customElement('product-card')
export class ProductCard extends LitElement {
  @property({ type: Number }) productId = 0;
  @property({ type: Object }) product: Product | null = null;
}

// types.d.ts (manual)
declare module 'react' {
  namespace JSX {
    interface IntrinsicElements {
      'product-card': React.DetailedHTMLProps<
        React.HTMLAttributes<ProductCard> & {
          productId?: number;
          product?: Product;
        },
        ProductCard
      >;
    }
  }
}
```

**Library `@types/...` packages:**

Mashhur Web Component library'lar TS types ta'minlaydi:

```typescript
// @shoelace-style/shoelace
import '@shoelace-style/shoelace/dist/components/button/button';
import type { SlButton } from '@shoelace-style/shoelace';

// @material/web
import '@material/web/button/filled-button';
import type { MdFilledButton } from '@material/web/button/filled-button';
```

<details>
<summary><strong>Kod Misollari</strong></summary>

To'liq custom element types file:

```typescript
// src/types/web-components.d.ts
import type { DetailedHTMLProps, HTMLAttributes } from 'react';

interface Product {
  id: number;
  name: string;
  price: number;
  description: string;
}

interface User {
  id: number;
  email: string;
}

interface ProductCardElement extends HTMLElement {
  product: Product | null;
  featured: boolean;
  onPurchase: ((productId: number) => void) | null;
}

interface UserAvatarElement extends HTMLElement {
  user: User | null;
  size: 'small' | 'medium' | 'large';
}

interface ProductCardAttributes extends HTMLAttributes<ProductCardElement> {
  product?: Product;
  featured?: boolean;
  onPurchase?: (productId: number) => void;
  'product-id'?: number;
}

interface UserAvatarAttributes extends HTMLAttributes<UserAvatarElement> {
  user?: User;
  size?: 'small' | 'medium' | 'large';
  alt?: string;
}

declare module 'react' {
  namespace JSX {
    interface IntrinsicElements {
      'product-card': DetailedHTMLProps<ProductCardAttributes, ProductCardElement>;
      'user-avatar': DetailedHTMLProps<UserAvatarAttributes, UserAvatarElement>;
    }
  }
}
```

```tsx
// Usage with full type safety
function ProductGrid({ products, currentUser }: {
  products: Product[];
  currentUser: User | null;
}) {
  return (
    <div className="grid">
      {currentUser && (
        <user-avatar user={currentUser} size="medium" />
      )}

      {products.map((product) => (
        <product-card
          key={product.id}
          product={product}
          featured={product.id === 1}
          onPurchase={(productId) => console.log('Buy:', productId)}
        />
      ))}
    </div>
  );
}
```

Generic Custom Element types helper:

```typescript
import type { DetailedHTMLProps, HTMLAttributes } from 'react';

export type CustomElementProps<
  TElement extends HTMLElement,
  TProperties = Record<string, never>,
  TEvents = Record<string, never>
> = DetailedHTMLProps<
  HTMLAttributes<TElement> & Partial<TProperties> & {
    [K in keyof TEvents as `on${Capitalize<string & K>}`]?: (e: TEvents[K]) => void;
  },
  TElement
>;

declare module 'react' {
  namespace JSX {
    interface IntrinsicElements {
      'my-list': CustomElementProps<
        HTMLElement,
        {
          items: Item[];
          selectedId: number | null;
        },
        {
          'item-select': CustomEvent<{ itemId: number }>;
        }
      >;
    }
  }
}
```

</details>

---

## Decision Guide — Web Component vs React Component

### Nazariya

Web Component vs React Component qarori — context-dependent. To'rt asosiy faktor:

**1. Cross-framework requirement:**

- **Web Component** — Vue, Svelte, Angular, vanilla JS, future framework'larda ishlatish.
- **React Component** — faqat React ekosistemasi ichida.

Misol: Design system company-wide → Web Component (har jamoa har xil framework). Single React app → React Component.

**2. Style isolation needs:**

- **Web Component (Shadow DOM)** — strict CSS isolation. Bashar style'lari komponentga ta'sir qilmaydi.
- **React Component** — CSS Modules / styled-components / Tailwind isolation, lekin global cascade ham mumkin.

**3. State complexity:**

- **Web Component** — class fields, manual reactivity (Lit `@property`, signals).
- **React Component** — hooks, Context, integration with state library'lari (Redux, Zustand, TanStack Query).

**4. Type safety:**

- **Web Component** — TS via decorators (Lit/Stencil), JSX types alohida deklaratsiya.
- **React Component** — TS first-class, props/state inference, end-to-end safety.

**Decision matrix:**

| Use case | Tavsiya |
|----------|---------|
| Design system cross-framework | Web Component |
| React-only app | React Component |
| Library author (universal) | Web Component (Lit) |
| Library author (React-only) | React Component |
| Existing React codebase migration | React Component |
| Microfrontend architecture | Web Component |
| Strict style isolation | Web Component |
| Complex state with library integration | React Component |
| Performance-critical (no framework overhead) | Web Component (native) |
| Server-side rendering | React Component (better support) |

**Hybrid approach:**

R19 React-WC interop yaxshi → mixed strategy:

- **Atomic primitives** — Web Component (Material Web, Shoelace).
- **Application logic** — React Component.
- **Cross-cutting widgets** — Web Component (analytics widget, ads).

<details>
<summary><strong>Kod Misollari</strong></summary>

Hybrid architecture pattern:

```tsx
import '@shoelace-style/shoelace/dist/components/button/button';
import '@shoelace-style/shoelace/dist/components/input/input';
import '@shoelace-style/shoelace/dist/components/dialog/dialog';

import { useState } from 'react';
import { useQuery, useMutation } from '@tanstack/react-query';

interface UserFormData {
  email: string;
  name: string;
}

function UserSettingsPage() {
  const [editing, setEditing] = useState(false);
  const [formData, setFormData] = useState<UserFormData>({ email: '', name: '' });

  const { data: user } = useQuery({
    queryKey: ['user', 'me'],
    queryFn: () => fetchCurrentUser(),
  });

  const updateMutation = useMutation({
    mutationFn: (data: UserFormData) => updateUser(data),
    onSuccess: () => setEditing(false),
  });

  if (!user) return <p>Loading...</p>;

  return (
    <div>
      <h1>Profile Settings</h1>

      {!editing ? (
        <div>
          <p>Email: {user.email}</p>
          <p>Name: {user.name}</p>

          <sl-button
            variant="primary"
            onClick={() => {
              setFormData({ email: user.email, name: user.name });
              setEditing(true);
            }}
          >
            Edit
          </sl-button>
        </div>
      ) : (
        <form
          onSubmit={(e) => {
            e.preventDefault();
            updateMutation.mutate(formData);
          }}
        >
          <sl-input
            label="Email"
            value={formData.email}
            onInput={(e: any) =>
              setFormData((prev) => ({ ...prev, email: e.target.value }))
            }
          />
          <sl-input
            label="Name"
            value={formData.name}
            onInput={(e: any) =>
              setFormData((prev) => ({ ...prev, name: e.target.value }))
            }
          />

          <sl-button type="submit" variant="primary" loading={updateMutation.isPending}>
            Save
          </sl-button>
          <sl-button onClick={() => setEditing(false)}>Cancel</sl-button>
        </form>
      )}
    </div>
  );
}
```

Pure Web Component approach (Lit):

```typescript
import { LitElement, html, css } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';

@customElement('user-settings')
export class UserSettings extends LitElement {
  @property({ type: Object })
  user: User | null = null;

  @state()
  private editing = false;

  @state()
  private formData: UserFormData = { email: '', name: '' };

  static styles = css`
    :host { display: block; }
    form { display: grid; gap: 16px; max-width: 400px; }
  `;

  render() {
    if (!this.user) return html`<p>Loading...</p>`;

    return this.editing
      ? this.renderEditForm()
      : this.renderDisplay();
  }

  private renderDisplay() {
    return html`
      <h1>Profile Settings</h1>
      <p>Email: ${this.user!.email}</p>
      <p>Name: ${this.user!.name}</p>
      <sl-button variant="primary" @click=${this.startEditing}>
        Edit
      </sl-button>
    `;
  }

  private renderEditForm() {
    return html`
      <form @submit=${this.handleSubmit}>
        <sl-input label="Email" .value=${this.formData.email}
                  @input=${(e: any) => this.updateField('email', e.target.value)}>
        </sl-input>
        <sl-input label="Name" .value=${this.formData.name}
                  @input=${(e: any) => this.updateField('name', e.target.value)}>
        </sl-input>
        <sl-button type="submit" variant="primary">Save</sl-button>
        <sl-button @click=${() => (this.editing = false)}>Cancel</sl-button>
      </form>
    `;
  }

  private startEditing = () => {
    this.formData = { email: this.user!.email, name: this.user!.name };
    this.editing = true;
  };

  private updateField = (key: keyof UserFormData, value: string) => {
    this.formData = { ...this.formData, [key]: value };
  };

  private handleSubmit = async (e: Event) => {
    e.preventDefault();
    await updateUser(this.formData);
    this.editing = false;
    this.dispatchEvent(new CustomEvent('user-updated'));
  };
}
```

</details>

---

## Edge Cases va Gotchas

### Custom Element Tag Name Defis Majburiy

Custom Element tag name'ida **kamida bitta defis** bo'lishi shart. Aks holda registratsiya fail.

```javascript
customElements.define('myelement', XCard);
// DOMException: Failed to execute 'define' on 'CustomElementRegistry':
// "myelement" is not a valid custom element name

customElements.define('my-element', XCard); // ✅ OK
```

### Reserved Tag Names

Quyidagi nomlar reserved:

```
annotation-xml, color-profile, font-face, font-face-src, font-face-uri,
font-face-format, font-face-name, missing-glyph
```

### Constructor'da DOM Manipulation Taqiqlanadi

```javascript
class XCard extends HTMLElement {
  constructor() {
    super();
    // ❌ TypeError: cannot manipulate before insertion
    // this.appendChild(...);
    // this.setAttribute('data-key', 'value');
  }
}
```

`connectedCallback`'ga ko'chirish kerak.

### Closed Shadow DOM va Testing

```javascript
class XCard extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'closed' });
  }
}

const element = document.querySelector('x-card');
console.log(element.shadowRoot); // null!
```

Closed mode testing va debugging'ni qiyinlashtiradi. Open mode afzal.

### React Synthetic Event vs Native Event Conflict

React `on*` props faqat ma'lum standart DOM event'lar uchun ishlaydi (`onClick`, `onChange`, `onInput`, `onFocus`, va h.k.) — React event delegation orqali. Custom Event'lar (`item-select`, `user-edit`) `on*` orqali tutilmaydi — `addEventListener` kerak.

Standart DOM event'lar (masalan `click`) Shadow DOM ichidan emit qilinsa, `composed: true` bo'lsa Light DOM'gacha bubble qiladi va React handler chaqiriladi:

```tsx
<custom-button
  onClick={() => console.log('React onClick — standard click event')}
/>

useEffect(() => {
  const el = ref.current;
  el?.addEventListener('click', () => console.log('Native click listener'));
}, []);

// Standard `click` event: ikkalasi ham trigger (React delegated + native listener)
// Custom event (e.g. 'item-select'): faqat addEventListener trigger
```

### SSR va Custom Elements

Custom Element'lar server'da React `renderToString` orqali shunchaki HTML tag sifatida emit qilinadi — class logic server'da execute qilinmaydi. Client'da JS yuklanguncha element "uninitialized" holatda turadi (`HTMLElement` instance). Class load bo'lib `customElements.define` chaqirilgandan keyin upgrade jarayoni boshlanadi.

Declarative Shadow DOM (`<template shadowrootmode="open">`) orqali server'da Shadow DOM markup ham yuborilishi mumkin (Chrome 111+, Safari 16.4+, Firefox 123+) — lekin bu pattern hali React'da to'g'ridan-to'g'ri qo'llab-quvvatlanmaydi. Workaround: `customElements.whenDefined` `await` qilish va Suspense ichida loading state.

### Hot Module Replacement Issues

Vite/Webpack HMR Custom Element redefinition'ni qo'llab-quvvatlamaydi (`customElements.define` ikkinchi marta chaqirsa throw). Workaround:

```typescript
if (!customElements.get('my-element')) {
  customElements.define('my-element', MyElement);
}
```

---

## Common Mistakes

### ❌ Xato 1: `onClick` o'rniga Custom Event handler

```tsx
// ❌ NOTO'G'RI — Custom event React on*'da ishlamaydi
<my-list onItemSelect={handleSelect} />

// ✅ TO'G'RI — useEffect + addEventListener
const ref = useRef<HTMLElement>(null);
useEffect(() => {
  const handler = (e: Event) => handleSelect((e as CustomEvent).detail);
  ref.current?.addEventListener('item-select', handler);
  return () => ref.current?.removeEventListener('item-select', handler);
}, [handleSelect]);
return <my-list ref={ref} />;
```

### ❌ Xato 2: TypeScript types yo'q

```tsx
// ❌ TS error: 'Property "my-card" does not exist on JSX.IntrinsicElements'
<my-card product={data} />

// ✅ TO'G'RI — JSX.IntrinsicElements augmentation
declare module 'react' {
  namespace JSX {
    interface IntrinsicElements {
      'my-card': React.DetailedHTMLProps<
        React.HTMLAttributes<HTMLElement> & { product?: Product },
        HTMLElement
      >;
    }
  }
}
```

### ❌ Xato 3: Pre-R19 attribute workaround R19'da

```tsx
// ❌ Pre-R19 workaround R19'da kerak emas
const ref = useRef<HTMLElement>(null);
useEffect(() => {
  if (ref.current) (ref.current as any).product = product;
}, [product]);
return <my-card ref={ref} />;

// ✅ TO'G'RI R19 — direct prop
return <my-card product={product} />;
```

### ❌ Xato 4: `customElements.define` ikki marta

```typescript
// ❌ HMR yoki double import — throw
customElements.define('my-element', MyElement);

// ✅ TO'G'RI — guard
if (!customElements.get('my-element')) {
  customElements.define('my-element', MyElement);
}
```

### ❌ Xato 5: Constructor'da DOM access

```typescript
// ❌ TypeError
class XCard extends HTMLElement {
  constructor() {
    super();
    this.appendChild(document.createElement('p')); // ❌ taqiq
  }
}

// ✅ TO'G'RI — connectedCallback
class XCard extends HTMLElement {
  connectedCallback() {
    this.appendChild(document.createElement('p'));
  }
}
```

---

## Amaliy Mashqlar

### Mashq 1: Custom Event Handler Hook (Oson)

`useCustomEvent` reusable hook yarating: event name, handler callback. Ref qaytaradi. Cleanup ta'minlaydi.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useEffect, useRef } from 'react';

function useCustomEvent<T = unknown>(
  eventName: string,
  handler: (detail: T) => void
) {
  const ref = useRef<HTMLElement>(null);
  const handlerRef = useRef(handler);
  handlerRef.current = handler;

  useEffect(() => {
    const element = ref.current;
    if (!element) return;

    const wrapped = (e: Event) => {
      handlerRef.current((e as CustomEvent<T>).detail);
    };

    element.addEventListener(eventName, wrapped);
    return () => element.removeEventListener(eventName, wrapped);
  }, [eventName]);

  return ref;
}

function ProductGrid({ onSelect }: { onSelect: (id: number) => void }) {
  const ref = useCustomEvent<{ productId: number }>(
    'product-select',
    ({ productId }) => onSelect(productId)
  );

  return <product-grid ref={ref} />;
}
```

**Tushuntirish:**

- `handlerRef` latest closure pattern (handler o'zgarsa re-attach kerak emas).
- `useEffect` deps `[eventName]` — eventName o'zgarsa re-attach.
- Type-safe via generic `T` (CustomEvent detail type).
- Cleanup `removeEventListener` shart.

</details>

---

### Mashq 2: Lit Element + R19 Properties (Oson)

Lit'da `user-card` element yarating (`name`, `email`, `avatar` properties + `select` event). R19 React'da ishlating.

<details>
<summary><strong>Javob</strong></summary>

```typescript
// user-card.ts
import { LitElement, html, css } from 'lit';
import { customElement, property } from 'lit/decorators.js';

interface User {
  id: number;
  name: string;
  email: string;
  avatar: string;
}

@customElement('user-card')
export class UserCard extends LitElement {
  static styles = css`
    :host {
      display: block;
      padding: 16px;
      border: 1px solid #e0e0e0;
      border-radius: 8px;
      cursor: pointer;
    }

    :host(:hover) {
      background: #f9f9f9;
    }

    .avatar {
      width: 48px;
      height: 48px;
      border-radius: 50%;
    }

    h3 { margin: 8px 0 4px; }
    p { margin: 0; color: #666; }
  `;

  @property({ type: Object })
  user: User | null = null;

  render() {
    if (!this.user) return html`<p>No user</p>`;

    return html`
      <div @click=${this.handleClick}>
        <img class="avatar" src=${this.user.avatar} alt=${this.user.name} />
        <h3>${this.user.name}</h3>
        <p>${this.user.email}</p>
      </div>
    `;
  }

  private handleClick = () => {
    this.dispatchEvent(
      new CustomEvent('user-select', {
        detail: { user: this.user },
        bubbles: true,
        composed: true,
      })
    );
  };
}
```

```tsx
// types.d.ts
import type { UserCard } from './user-card';

declare module 'react' {
  namespace JSX {
    interface IntrinsicElements {
      'user-card': React.DetailedHTMLProps<
        React.HTMLAttributes<UserCard> & { user?: User },
        UserCard
      >;
    }
  }
}

// React component:
import './user-card';
import { useCustomEvent } from './hooks/useCustomEvent';

function UsersList({ users, onUserSelect }: {
  users: User[];
  onUserSelect: (user: User) => void;
}) {
  const ref = useCustomEvent<{ user: User }>(
    'user-select',
    ({ user }) => onUserSelect(user)
  );

  return (
    <div ref={ref} className="users-list">
      {users.map((user) => (
        <user-card key={user.id} user={user} />
      ))}
    </div>
  );
}
```

**Tushuntirish:**

- Lit `@property({ type: Object })` — User obyekt direct binding.
- R19 React `user={user}` — automatic property assignment (object detection).
- Custom event `composed: true` — Shadow DOM boundary'ni kesib o'tadi.
- `useCustomEvent` hook — event listener boilerplate'siz.
- TS JSX intrinsic elements augmentation — type safety.

</details>

---

### Mashq 3: Web Component Wrapper React Component (O'rta)

`MyButton` React komponenti yarating, ichida Lit `<sl-button>` (Shoelace). Props pass qiling, click handler, loading state.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import '@shoelace-style/shoelace/dist/components/button/button';
import { ReactNode, MouseEvent } from 'react';

declare module 'react' {
  namespace JSX {
    interface IntrinsicElements {
      'sl-button': React.DetailedHTMLProps<
        React.HTMLAttributes<HTMLElement> & {
          variant?: 'default' | 'primary' | 'success' | 'neutral' | 'warning' | 'danger' | 'text';
          size?: 'small' | 'medium' | 'large';
          loading?: boolean;
          disabled?: boolean;
          type?: 'button' | 'submit' | 'reset';
        },
        HTMLElement
      >;
    }
  }
}

interface MyButtonProps {
  variant?: 'primary' | 'danger' | 'default';
  size?: 'small' | 'medium' | 'large';
  loading?: boolean;
  disabled?: boolean;
  type?: 'button' | 'submit';
  onClick?: (e: MouseEvent<HTMLElement>) => void;
  children: ReactNode;
}

function MyButton({
  variant = 'default',
  size = 'medium',
  loading = false,
  disabled = false,
  type = 'button',
  onClick,
  children,
}: MyButtonProps) {
  return (
    <sl-button
      variant={variant}
      size={size}
      loading={loading}
      disabled={disabled}
      type={type}
      onClick={onClick}
    >
      {children}
    </sl-button>
  );
}

function CheckoutPage() {
  const [submitting, setSubmitting] = useState(false);

  const handleSubmit = async () => {
    setSubmitting(true);
    try {
      await processOrder();
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <form>
      <MyButton variant="default" type="button">Cancel</MyButton>
      <MyButton
        variant="primary"
        type="submit"
        loading={submitting}
        disabled={submitting}
        onClick={handleSubmit}
      >
        Place Order
      </MyButton>
    </form>
  );
}
```

**Tushuntirish:**

- `JSX.IntrinsicElements` augmentation — `<sl-button>` type safety.
- `MyButton` props — restricted variant set (Shoelace'da ko'p, lekin app'da 3 ishlatamiz).
- R19 React'da `loading`, `disabled` boolean prop'lar to'g'ri set qilinadi.
- `onClick` React synthetic event — Shoelace `<sl-button>` standard `click` event emit qiladi.
- `children` Light DOM'ga inject qilinadi.

</details>

---

### Mashq 4: TypeScript Generic Custom Element (O'rta)

Generic `<DataList>` Custom Element wrapper yarating: items array, item render template, selection event. TypeScript strict types.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useRef, useEffect, ReactNode } from 'react';

interface ItemSelectEvent<T> extends CustomEvent<{ item: T; index: number }> {}

interface DataListElement<T> extends HTMLElement {
  items: T[];
  renderItem: ((item: T, index: number) => string) | null;
}

declare module 'react' {
  namespace JSX {
    interface IntrinsicElements {
      'data-list': React.DetailedHTMLProps<
        React.HTMLAttributes<DataListElement<unknown>> & {
          items?: unknown[];
          renderItem?: (item: unknown, index: number) => string;
        },
        DataListElement<unknown>
      >;
    }
  }
}

interface DataListProps<T> {
  items: T[];
  renderItem: (item: T, index: number) => string;
  onItemSelect?: (item: T, index: number) => void;
  emptyState?: ReactNode;
}

function DataList<T>({
  items,
  renderItem,
  onItemSelect,
  emptyState,
}: DataListProps<T>) {
  const ref = useRef<DataListElement<T>>(null);
  const callbacksRef = useRef({ renderItem, onItemSelect });
  callbacksRef.current = { renderItem, onItemSelect };

  useEffect(() => {
    const element = ref.current;
    if (!element) return;

    element.items = items;
    element.renderItem = (item, index) =>
      callbacksRef.current.renderItem(item as T, index);
  }, [items]);

  useEffect(() => {
    const element = ref.current;
    if (!element) return;

    const handler = (e: Event) => {
      const customEvent = e as ItemSelectEvent<T>;
      callbacksRef.current.onItemSelect?.(
        customEvent.detail.item,
        customEvent.detail.index
      );
    };

    element.addEventListener('item-select', handler);
    return () => element.removeEventListener('item-select', handler);
  }, []);

  if (items.length === 0 && emptyState) {
    return <>{emptyState}</>;
  }

  return <data-list ref={ref as React.Ref<DataListElement<unknown>>} />;
}

interface Product {
  id: number;
  name: string;
  price: number;
}

function ProductsListPage({ products }: { products: Product[] }) {
  return (
    <DataList<Product>
      items={products}
      renderItem={(product) =>
        `<div><h3>${product.name}</h3><p>$${product.price}</p></div>`
      }
      onItemSelect={(product) => console.log('Selected:', product)}
      emptyState={<p>No products available</p>}
    />
  );
}
```

**Tushuntirish:**

- Generic `<T>` type parameter — DataList har turdagi item bilan ishlaydi.
- `items` va `renderItem` Custom Element property assignment (R19 native).
- `onItemSelect` Custom Event listener (React on* ishlamaydi).
- `callbacksRef` latest closure pattern — re-attach kerak emas.
- Optional `emptyState` React fallback content.

</details>

---

### Mashq 5: Production Mixed Architecture (Qiyin)

Hybrid architecture yarating: React app + Lit Web Components + TanStack Query. UserDashboard (React) + UserCard (Lit) + Custom Events + R19 properties + TS types.

<details>
<summary><strong>Javob</strong></summary>

```typescript
// components/user-card.ts
import { LitElement, html, css } from 'lit';
import { customElement, property } from 'lit/decorators.js';

export interface UserCardData {
  id: number;
  name: string;
  email: string;
  avatar: string;
  role: 'admin' | 'user' | 'moderator';
  status: 'active' | 'pending' | 'banned';
}

@customElement('user-card')
export class UserCardElement extends LitElement {
  static styles = css`
    :host {
      display: block;
      padding: 16px;
      border: 1px solid var(--border-color, #e0e0e0);
      border-radius: 8px;
      background: var(--card-bg, white);
    }

    :host([status="banned"]) {
      opacity: 0.5;
    }

    .header {
      display: flex;
      align-items: center;
      gap: 12px;
    }

    .avatar {
      width: 48px;
      height: 48px;
      border-radius: 50%;
      object-fit: cover;
    }

    .name {
      font-weight: 600;
      margin: 0;
    }

    .email {
      color: #666;
      margin: 0;
      font-size: 14px;
    }

    .badge {
      display: inline-block;
      padding: 2px 8px;
      border-radius: 4px;
      font-size: 12px;
      background: #f0f0f0;
    }

    .badge.admin { background: #fee; color: #c00; }
    .badge.moderator { background: #ffe; color: #aa0; }

    .actions {
      margin-top: 12px;
      display: flex;
      gap: 8px;
    }

    button {
      padding: 6px 12px;
      border: none;
      border-radius: 4px;
      cursor: pointer;
    }

    button.primary {
      background: var(--primary-color, #0066cc);
      color: white;
    }

    button.danger {
      background: #c00;
      color: white;
    }
  `;

  @property({ type: Object })
  user: UserCardData | null = null;

  @property({ type: Boolean, reflect: true })
  loading = false;

  render() {
    if (!this.user) return html`<p>Loading...</p>`;

    return html`
      <div class="header">
        <img class="avatar" src=${this.user.avatar} alt=${this.user.name} />
        <div>
          <p class="name">${this.user.name}</p>
          <p class="email">${this.user.email}</p>
          ${this.user.role !== 'user'
            ? html`<span class="badge ${this.user.role}">${this.user.role}</span>`
            : null}
        </div>
      </div>

      <div class="actions">
        <button class="primary" @click=${this.handleEdit} ?disabled=${this.loading}>
          Edit
        </button>
        <button class="danger" @click=${this.handleDelete} ?disabled=${this.loading}>
          Delete
        </button>
      </div>
    `;
  }

  private handleEdit = () => {
    this.dispatchEvent(
      new CustomEvent('user-edit', {
        detail: { user: this.user },
        bubbles: true,
        composed: true,
      })
    );
  };

  private handleDelete = () => {
    this.dispatchEvent(
      new CustomEvent('user-delete', {
        detail: { userId: this.user?.id },
        bubbles: true,
        composed: true,
      })
    );
  };
}
```

```tsx
// hooks/useCustomEvents.ts
import { useEffect, useRef } from 'react';

export function useCustomEvents<T extends Record<string, unknown>>(
  handlers: { [K in keyof T]?: (detail: T[K]) => void }
) {
  const ref = useRef<HTMLElement>(null);
  const handlersRef = useRef(handlers);
  handlersRef.current = handlers;

  const eventKeys = Object.keys(handlersRef.current);

  useEffect(() => {
    const element = ref.current;
    if (!element) return;

    const subs = eventKeys.map((eventName) => {
      const wrapper = (e: Event) => {
        const handler = handlersRef.current[eventName as keyof T];
        handler?.((e as CustomEvent).detail);
      };
      element.addEventListener(eventName, wrapper);
      return () => element.removeEventListener(eventName, wrapper);
    });

    return () => subs.forEach((unsub) => unsub());
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [eventKeys.join('|')]);

  return ref;
}
```

```tsx
// pages/UserDashboard.tsx
import '../components/user-card';
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { useState } from 'react';
import { useCustomEvents } from '../hooks/useCustomEvents';
import type { UserCardData } from '../components/user-card';

interface UserEvents {
  'user-edit': { user: UserCardData };
  'user-delete': { userId: number };
}

function UserDashboard() {
  const queryClient = useQueryClient();
  const [editingUserId, setEditingUserId] = useState<number | null>(null);

  const { data: users = [], isLoading } = useQuery<UserCardData[]>({
    queryKey: ['users'],
    queryFn: () => fetch('/api/users').then((r) => r.json()),
  });

  const deleteMutation = useMutation({
    mutationFn: (userId: number) =>
      fetch(`/api/users/${userId}`, { method: 'DELETE' }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });

  const ref = useCustomEvents<UserEvents>({
    'user-edit': ({ user }) => {
      setEditingUserId(user.id);
    },
    'user-delete': ({ userId }) => {
      if (confirm('O\'chirishni tasdiqlaysizmi?')) {
        deleteMutation.mutate(userId);
      }
    },
  });

  if (isLoading) return <div>Loading users...</div>;

  return (
    <div className="user-dashboard">
      <h1>Users ({users.length})</h1>

      <div ref={ref} className="users-grid">
        {users.map((user) => (
          <user-card
            key={user.id}
            user={user}
            loading={
              deleteMutation.isPending && deleteMutation.variables === user.id
            }
          />
        ))}
      </div>

      {editingUserId !== null && (
        <UserEditModal
          userId={editingUserId}
          onClose={() => setEditingUserId(null)}
        />
      )}
    </div>
  );
}
```

**Tushuntirish:**

- **Lit UserCard** — Shadow DOM CSS isolation, reactive properties, custom events.
- **R19 React** — `user={user}` direct property assignment, `loading={true}` boolean prop.
- **TS JSX intrinsic elements** — full type safety.
- **`useCustomEvents` hook** — multiple events bitta hook.
- **TanStack Query integration** — server state management.
- **Mixed responsibilities** — Web Component visual presentation, React app state va orchestration.
- **CSS variables** — `--primary-color`, `--card-bg` outer theme'dan inject qilinadi.

</details>

---

## Xulosa

- **Web Components** — to'rtta brauzer-native API: Custom Elements, Shadow DOM, HTML Templates, Custom Events. Framework-agnostic UI komponent'lar yaratish.
- **Custom Element registratsiyasi** — `customElements.define('name', Class)`. Tag name'da defis majburiy. Constructor'da DOM manipulation taqiq, `connectedCallback` ishlatish.
- **Lifecycle callbacks** — `connectedCallback` (mount), `disconnectedCallback` (unmount), `attributeChangedCallback` (observed attribute), `adoptedCallback` (rare).
- **Pre-R19 Properties vs Attributes muammo** — React har JSX prop'ni HTML attribute sifatida set qilardi → object/function/array `[object Object]` string'ga konvertatsiya. Manual `useEffect + ref` workaround.
- **R19 yechimi** — primitive value attribute, complex value (`object`/`function`/`array`/`boolean`) DOM property assignment. `propName in element` check ham (Lit reactive properties uchun).
- **Custom Events React'da** — `on*` props ishlamaydi (synthetic event tizimi faqat ma'lum DOM event'larni qo'llab-quvvatlaydi). `useEffect` + `addEventListener` pattern. Reusable `useCustomEvent` hook.
- **Shadow DOM** — CSS va DOM isolation. `:host`, `::slotted()`, `:host-context()` selectors. CSS variables Shadow DOM boundary'ni kesib o'tadi.
- **Slots** — Light DOM children'ni Shadow DOM ichidagi `<slot>` joylariga inject. Default slot, named slots, fallback content.
- **React Component Web Component ichida** — `createRoot` `connectedCallback`, `unmount` `disconnectedCallback`. Cross-framework integration uchun foydali.
- **TypeScript JSX Intrinsic Elements** — `declare module 'react' { namespace JSX { interface IntrinsicElements {...} } }` augmentation. Library'lar (`@shoelace-style/shoelace`, `@material/web`) types ta'minlaydi.
- **Decision Guide:** Cross-framework → Web Component; React-only app → React Component; Hybrid mature → atomic Web Component + React orchestration.
- **Anti-pattern'lar:** `onClick` o'rniga Custom Event kutish, TS types yo'q, Pre-R19 workaround R19'da, `customElements.define` ikki marta, constructor'da DOM access.

---

**Keyingi bo'lim:** [39-rsc-server-actions.md](39-rsc-server-actions.md) — Kursning yakuniy bo'limi: React Server Components (RSC) concept (server'da render, client'ga HTML+RSC payload), `'use server'` va `'use client'` directives, Server vs Client component farqi, Server Actions (`useActionState` + `useOptimistic` integration cross-ref 23), `cache(fn)` per-request memoization, Streaming SSR + RSC, framework cheklov (Next.js App Router, TanStack Start vanilla React'da yo'q), implementation `/next/` kursida ko'riladi.
