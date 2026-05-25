# Bo'lim 8: List Rendering va Keys

> List rendering — array ma'lumotini JSX element'lar yig'masiga o'tkazish jarayoni. `key` prop esa Reconciliation algorithmiga node identity'ni xabarlash mexanizmi. Bu bo'lim `map()` pattern'ini, `key` qoidalarini, index-as-key trade-off'ini, nested list'larni va kalit-asosli komponent state hayotiyligini yoritadi.

---

## Mundarija

- [Lists va JSX — `map()` Pattern](#lists-va-jsx--map-pattern)
- [Nima uchun `key` Talab Qilinadi](#nima-uchun-key-talab-qilinadi)
- [`key` Prop Qoidalari](#key-prop-qoidalari)
- [Index `key` Sifatida — Trade-off](#index-key-sifatida--trade-off)
- [`key` va Komponent Identity](#key-va-komponent-identity)
- [Nested Lists va Key Scope](#nested-lists-va-key-scope)
- [Key Collisions va Composite Keys](#key-collisions-va-composite-keys)
- [Reordering Performance — Reconciler Bog'lanishi](#reordering-performance--reconciler-boglanishi)
- [Fragment va Special Cases](#fragment-va-special-cases)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Lists va JSX — `map()` Pattern

### Nazariya

JSX **expression** sifatida ishlaydi (`07-jsx.md` Section: "JSX = JS expression"). Curly braces `{}` ichida JS qiymat — string, number, element, **element array** — render qilinadi. Shu sababli array → JSX render uchun maxsus syntax kerak emas: array'ni JSX element'lar bilan to'ldirib, uni curly braces ichiga qo'yiladi.

`Array.prototype.map()` bu uchun idiomatik vosita. `map` har element uchun yangi qiymat (JSX element) qaytaradi va natija — element array. JSX bu array'ni iterativ ravishda render qiladi.

```tsx
type Product = { id: number; name: string; price: number };

const products: Product[] = [
  { id: 1, name: 'Keyboard', price: 49 },
  { id: 2, name: 'Mouse', price: 19 },
];

function ProductList() {
  return (
    <ul>
      {products.map((product) => (
        <li key={product.id}>
          {product.name} — ${product.price}
        </li>
      ))}
    </ul>
  );
}
```

`map` callback'ning return qiymati `ReactElement`. TypeScript bu type'ni `ReactNode` array sifatida tan oladi. JSX engine array'ni traverse qilib, har element'ni alohida child sifatida ishlov beradi.

`forEach` yoki klassik `for` loop bu contextda mos kelmaydi — ular qiymat qaytarmaydi (`undefined`), JSX render qila olmaydi. `map` esa **transform va return** qiladi, declarative approach'ga to'g'ri keladi.

`filter`, `slice`, `reduce` ham natija sifatida array berishi mumkin va shu kabi mexanizm bilan ishlaydi:

```tsx
{products
  .filter((p) => p.price < 30)
  .map((p) => <li key={p.id}>{p.name}</li>)
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

JSX transform array'ni JS array sifatida saqlaydi. Quyidagi JSX:

```tsx
<ul>
  {products.map((p) => <li key={p.id}>{p.name}</li>)}
</ul>
```

Avtomatik runtime (R17+) tomonidan quyidagicha o'zgartiriladi:

```tsx
import { jsx as _jsx } from 'react/jsx-runtime';

_jsx('ul', {
  children: products.map((p) =>
    _jsx('li', { children: p.name }, p.id)
  )
});
```

Diqqat qiling: `_jsx`'ning **uchinchi argumenti** — `key`. JSX transform `key` prop'ni `props` object'idan ajratib oladi va alohida positional argument sifatida uzatadi. Bu — `key`'ning React Element internal slot'ida (`element.key`) saqlanishini ta'minlaydi va u **prop sifatida komponentga uzatilmaydi**.

`_jsx` vs `_jsxs` farqi — **source-level child count** asosida:
- `_jsx` — manbada **bitta child expression** (JSX'da `<ul>{products.map(...)}</ul>` — bitta `{}` expression, runtime'da array bersa ham)
- `_jsxs` — manbada **bir nechta static child element** yonma-yon (`<ul><li/><li/></ul>` — transformer kompilatsiya paytida ko'radi)

Bu farq dev-mode key validation behaviour'iga ta'sir qiladi: `_jsxs` static children pozitsiyon stable bo'lgani uchun per-child key warning chiqarmaydi; `_jsx`'ga uzatilgan dynamic array (`map` natijasi) uchun React runtime per-element `key` tekshiruvi yoqiladi.

React bu element array'ni Reconciliation paytida ishlatadi:

1. Render Phase'da `reconcileChildrenArray(returnFiber, currentFirstChild, newChildren, lanes)` chaqiriladi (cross-ref [`04-reconciliation.md`](04-reconciliation.md) — Sibling Matching Keyed bo'limi)
2. React har element'ning `key` slot'ini o'qib, eski Fiber'lar bilan moslashtiradi
3. `key` mos kelsa — eski Fiber qayta ishlatiladi (state, refs, DOM saqlanadi)
4. `key` mos kelmasa — yangi Fiber yaratiladi, eski unmount qilinadi

```
Element array (newChildren):
[
  { type: 'li', key: '1', props: {...} },
  { type: 'li', key: '2', props: {...} },
  ...
]

Old Fiber chain:
li(key=1) → li(key=2) → li(key=3) → null
              │
              ▼
   Reconciler key map orqali eshlashtiradi
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Sodda misol — order list'i:

```tsx
type Order = { id: string; total: number; status: 'pending' | 'shipped' | 'delivered' };

const orders: Order[] = [
  { id: 'ord_001', total: 120, status: 'shipped' },
  { id: 'ord_002', total: 85, status: 'pending' },
  { id: 'ord_003', total: 210, status: 'delivered' },
];

function OrderTable() {
  return (
    <table>
      <thead>
        <tr>
          <th>ID</th>
          <th>Total</th>
          <th>Status</th>
        </tr>
      </thead>
      <tbody>
        {orders.map((order) => (
          <tr key={order.id}>
            <td>{order.id}</td>
            <td>${order.total}</td>
            <td>{order.status}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

Filter + map zanjiri — faqat aktiv user'lar:

```tsx
type User = { id: number; name: string; active: boolean };

function ActiveUserList({ users }: { users: User[] }) {
  return (
    <ul>
      {users
        .filter((user) => user.active)
        .map((user) => (
          <li key={user.id}>{user.name}</li>
        ))}
    </ul>
  );
}
```

Map bilan inline component — `tbody` ichida `Row` komponentni ishlatish:

```tsx
type Task = { id: number; title: string; done: boolean };

function TaskRow({ task }: { task: Task }) {
  return (
    <tr>
      <td>{task.title}</td>
      <td>{task.done ? '✓' : '○'}</td>
    </tr>
  );
}

function TaskList({ tasks }: { tasks: Task[] }) {
  return (
    <table>
      <tbody>
        {tasks.map((task) => (
          <TaskRow key={task.id} task={task} />
          // key TaskRow'ga uzatilmaydi — React uni ushlab qoladi
        ))}
      </tbody>
    </table>
  );
}
```

Anti-pattern: `forEach` bilan render urinish:

```tsx
// ❌ Ishlamaydi — forEach undefined qaytaradi
function BadList({ items }: { items: string[] }) {
  return (
    <ul>
      {items.forEach((item) => (
        <li>{item}</li>
      ))}
    </ul>
  );
  // Output: <ul></ul> — list bo'sh
}

// ✅ To'g'ri — map qiymat qaytaradi
function GoodList({ items }: { items: string[] }) {
  return (
    <ul>
      {items.map((item, index) => (
        <li key={index}>{item}</li>
      ))}
    </ul>
  );
}
```

Reduce bilan grouped rendering:

```tsx
type Product = { id: number; category: string; name: string };

function GroupedProducts({ products }: { products: Product[] }) {
  const grouped = products.reduce<Record<string, Product[]>>((acc, p) => {
    (acc[p.category] ??= []).push(p);
    return acc;
  }, {});

  return (
    <div>
      {Object.entries(grouped).map(([category, items]) => (
        <section key={category}>
          <h3>{category}</h3>
          <ul>
            {items.map((item) => (
              <li key={item.id}>{item.name}</li>
            ))}
          </ul>
        </section>
      ))}
    </div>
  );
}
```

</details>

---

## Nima uchun `key` Talab Qilinadi

### Nazariya

`key` prop — React'ning Reconciliation algorithmi uchun **identity belgi**. Bu — array bo'lgan child'lar orasida "qaysi yangi element qaysi eskisiga mos keladi" savoliga aniq javob beradigan unique tag.

`key` bo'lmasa, React **index-based matching** ishlatadi — ya'ni eski va yangi child array'larini positionga ko'ra solishtiradi (`children[0]` bilan `children[0]`, `children[1]` bilan `children[1]`, va h.k.). Bu — append (oxiriga qo'shish) holatida ishlaydi, lekin **prepend** (boshiga qo'shish), **insert** (orasiga kiritish), yoki **reorder** (qayta tartiblash) holatlarida noto'g'ri natijaga olib keladi.

Misol — array boshiga element qo'shilgan:

```
Eski: [A, B, C]
Yangi: [Z, A, B, C]

Index-based matching natijasi:
  Position 0: A → Z (state/DOM updated)
  Position 1: B → A (state/DOM updated)
  Position 2: C → B (state/DOM updated)
  Position 3: (yangi) → C (yangi yaratildi)

Natija: 4 ta o'zgarish.
```

Endi `key` bilan bir xil holat:

```
Eski: [A(key=a), B(key=b), C(key=c)]
Yangi: [Z(key=z), A(key=a), B(key=b), C(key=c)]

Keyed matching natijasi:
  key=z: yangi yaratildi (insert)
  key=a, key=b, key=c: eski Fiber qayta ishlatildi (state, DOM saqlandi)

Natija: 1 ta DOM insertion.
```

Bu — performance va correctness farqi. Performance — kam DOM operation. Correctness — input value, focus, scroll position, animation state, ref'lar — barchasi to'g'ri item bilan birga "yuradi".

`key` qiymati `string | number | bigint`. JSX transform `key` ni element'ning maxsus internal slot'ga (`ReactElement.key`) saqlaydi va `props` object'idan ajratib qo'yadi. Shuning uchun komponent ichida `props.key` mavjud emas — `key` faqat React Reconciler uchun.

> **`key` xatti-harakati — React versiyalari bo'ylab stable:**
> - `key` uzatilmasa, dev mode'da console warning chiqadi: `Warning: Each child in a list should have a unique "key" prop`. R16+ versiyalarining barchasida bu warning mavjud.
> - `key` hech qachon komponent prop'i sifatida uzatilmaydi — JSX transform `props` object'idan ajratib, `_jsx(type, props, key)` chaqirig'idagi uchinchi argument sifatida saqlaydi.
> - **Sabab:** `key` semantik jihatdan oddiy prop emas — Reconciler uchun internal identity signali. Komponentga uzatilsa, prop sifatida noto'g'ri ishlatish (`<Item key={x}>` ichida `props.key` o'qish urinishi) ehtimoli oshardi.

<details>
<summary><strong>Under the Hood</strong></summary>

`reconcileChildrenArray` algorithmining 2 bosqichi (cross-ref [`04-reconciliation.md`](04-reconciliation.md) — Sibling Matching Keyed va `lastPlacedIndex`):

**1-faza (sequential):** Yangi va eski child'larning birinchi `key` mos kelmagan positiongacha ketma-ket walk qilinadi. Bu faza — append/prepend bo'lmagan oddiy update'larni tezda hal qilish uchun.

**2-faza (key map):** Mos kelmagan positiondan boshlab, qolgan eski child'lar `Map<key, Fiber>` ga to'planadi. Yangi child'lar bu Map'dan `key` orqali izlanadi — topilsa, eski Fiber qayta ishlatiladi (`alternate` mexanizmi orqali); topilmasa — yangi Fiber yaratiladi.

```
Yangi child'lar walking pointer →
                ▼
[..., Z, A, B, C]
              ▲
              │ Map lookup: key=A → eski Fiber A
              │             key=B → eski Fiber B
              │             key=C → eski Fiber C
              │             key=Z → null (yangi)
              ▼
Eski child'lar Map: { 'a': Fiber_A, 'b': Fiber_B, 'c': Fiber_C }
```

**`lastPlacedIndex`** — greedy move detection algorithm:

- Har bir matched eski Fiber uchun, uning eski positionsi `lastPlacedIndex`'dan kichik bo'lsa, "move" flag qo'yiladi (DOM'da `insertBefore` chaqiriladi)
- Aks holda, `lastPlacedIndex` yangilanadi va Fiber joyida qoladi

Bu greedy algorithm Longest Common Subsequence (LCS) optimal emas, lekin O(n) tezligida ishlaydi. Practical reorder operationlari uchun yetarli.

**Internal helper'lar:**
- `createFiberFromElement(element, lanes)` — yangi Fiber yaratish
- `useFiber(currentFiber, pendingProps)` — eski Fiber'ni yangi props bilan qayta ishlatish
- `placeChild(newFiber, lastPlacedIndex, newIndex)` — flag qo'yish (`Placement`)

`key` taqqoslash reference (`Object.is`) emas, **string equality** asosida bo'ladi. Internal kod (taxminan):

```ts
function matchChild(oldFiber: Fiber | null, newChild: ReactElement): boolean {
  if (oldFiber === null) return false;
  const newKey = newChild.key;
  return oldFiber.key === newKey;
}
```

`key` `null` bo'lsa (JSX'da `key` berilmagan default holat), Reconciler **index-based positionviy matching**'ga o'tadi — yangi va eski child'lar position bo'yicha (1-1, 2-2, ...) eshlashtiriladi. `key` `null` qiymati Map ichida `null` sifatida saqlanmaydi; o'rniga `existingChildren.set(existingChild.index, existingChild)` (numeric index Map ichida) ishlatiladi. Boshqacha qilib aytganda, key'siz element'ning "key"i — uning positional index'i.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`key`siz vs `key` bilan — focus saqlanishi:

```tsx
import { useState } from 'react';

type Comment = { id: number; text: string };

function CommentList() {
  const [comments, setComments] = useState<Comment[]>([
    { id: 1, text: 'First' },
    { id: 2, text: 'Second' },
  ]);

  const prepend = () => {
    setComments((prev) => [
      { id: Date.now(), text: 'Newest' },
      ...prev,
    ]);
  };

  return (
    <div>
      <button onClick={prepend}>Add at top</button>
      <ul>
        {comments.map((c) => (
          <li key={c.id}>
            {/* ✅ key bilan: input focus va value to'g'ri item bilan qoladi */}
            <input defaultValue={c.text} />
          </li>
        ))}
      </ul>
    </div>
  );
}
```

`key`siz holat (xato) — input value yangi item'ga "siljib" ketadi:

```tsx
function BadCommentList({ comments }: { comments: Comment[] }) {
  return (
    <ul>
      {comments.map((c) => (
        // ❌ key yo'q — index-based matching, prepend buzadi
        <li>
          <input defaultValue={c.text} />
        </li>
      ))}
    </ul>
  );
  // Foydalanuvchi 1-input'ga 'Hello' yozgan
  // Yangi comment qo'shildi (top'ga)
  // Endi 'Hello' yangi top item'da ko'rinadi (xato!)
}
```

`key` komponentga prop sifatida uzatilmaydi:

```tsx
type ItemProps = { id: number; label: string };

function Item({ id, label }: ItemProps) {
  // props.key bu yerda mavjud EMAS
  // Agar id kerak bo'lsa, alohida prop sifatida uzatiladi
  return <li>{label} (#{id})</li>;
}

function ItemList({ items }: { items: ItemProps[] }) {
  return (
    <ul>
      {items.map((item) => (
        <Item key={item.id} id={item.id} label={item.label} />
        //    ↑ React uchun     ↑ Item komponentga uzatiladi
      ))}
    </ul>
  );
}
```

</details>

---

## `key` Prop Qoidalari

### Nazariya

`key` qiymati 4 ta talabni bajarishi shart. Ulardan biron bittasi buzilsa — Reconciliation noto'g'ri identity matching qiladi va component state, DOM, ref'lar bilan bog'liq xatolar paydo bo'ladi.

**Qoida 1 — Unique within siblings:** `key` faqat **bir parent ichidagi sibling'lar** orasida unique bo'lishi shart. Boshqa parent'larda yoki butun document'da unique bo'lishi talab qilinmaydi. Bu — Reconciler'ning sibling-level matching pattern'idan kelib chiqadi.

```tsx
// ✅ OK — har <ul> alohida scope
<div>
  <ul>
    <li key="1">A</li>
    <li key="2">B</li>
  </ul>
  <ul>
    <li key="1">X</li>  {/* OK — boshqa parent ichida */}
    <li key="2">Y</li>
  </ul>
</div>
```

**Qoida 2 — Stable across renders:** Bir item bir xil `key` ga ega bo'lishi kerak — qancha render bo'lsa ham. Agar item `[A, B, C]` da `key="a", key="b", key="c"` bo'lsa, keyingi render'da ham shunday bo'lishi kerak. Bu — Reconciler'ning eski va yangi Fiber'larni eshlashtirishi uchun yagona signal.

**Qoida 3 — Predictable:** `key` deterministic bo'lishi kerak — bir xil input → bir xil output. `Math.random()`, `Date.now()`, va boshqa har render'da yangi qiymat beradigan funksiyalar `key` sifatida **TAQIQ**.

```tsx
// ❌ Har render'da yangi key — har item butunlay qayta yaratiladi
{items.map((item) => (
  <Item key={Math.random()} item={item} />
))}
// Natija: state yo'qoladi, focus yo'qoladi, animation buziladi
```

**Qoida 4 — Tip cheklovi:** `key` qiymati `string | number | bigint`. Boolean, object, function, undefined, null **TAQIQ**. React qiymatni internal'da `'' + key` orqali string'ga o'tkazadi.

```tsx
// ❌ Object — toString() = "[object Object]" — barcha key bir xil bo'ladi
<Item key={item} />

// ❌ Boolean — har doim "true" yoki "false"
<Item key={item.active} />

// ✅ String yoki number
<Item key={item.id} />
```

**Yaxshi `key` manbalari:**

- ✅ Database'dan kelgan ID (`item.id`, `item.uuid`)
- ✅ Server-tomonida generation qilingan key
- ✅ Stable composite key (`${userId}_${productId}`)
- ✅ Item yaratilganda bir marta generation qilingan UUID (`crypto.randomUUID()` — saqlanadi va qayta ishlatiladi)

**Yomon `key` manbalari:**

- ❌ Array index dynamic list'larda
- ❌ `Math.random()`, `Date.now()` har render'da
- ❌ Object reference yoki har render'da yangi obyekt

<details>
<summary><strong>Under the Hood</strong></summary>

React `key` qiymatini internal jarayonda `KeyedFragment` yoki Fiber `key` slot'iga saqlaydi. Reconciler taqqoslashda reference emas, **string equality** ishlatadi:

```ts
// React internal (soddalashtirilgan)
function sameKey(oldFiber: Fiber, newElement: ReactElement): boolean {
  return oldFiber.key === newElement.key;
}
```

Numeric va string key bir xil bo'lsa qaytadi:

```tsx
{items.map((item, i) => <li key={i}>...</li>)}
{items.map((item, i) => <li key={String(i)}>...</li>)}
// Ikkalasi ham bir xil — React internal tomonida string'ga konvert qiladi
```

`bigint` key — React internal'da `String(key)` orqali konvert qilinadi (TypeScript'ning `Key` tipida `string | number | bigint` qabul qilinadi). Versiya cheklovi yo'q — `bigint` qo'llab-quvvatlash JS native `String(BigInt)` operation'iga asoslangan:

```tsx
{items.map((item) => <li key={item.bigId}>...</li>)}
// item.bigId: bigint — internal'da String konvert orqali ishlaydi
```

ESLint plugin `eslint-plugin-react`'ning `react/jsx-key` qoidasi `map`, `forEach` va boshqa array iteration callback'larida `key` mavjudligini static analysis bilan tekshiradi. Bu — runtime warning'dan oldin xatoni ushlash imkonini beradi.

Dev mode'da React har render'da quyidagi tekshiruvlarni amalga oshiradi:
1. Har element'ning `key` prop'i mavjudmi
2. Sibling'lar orasida duplikat `key` borligini
3. `key` `undefined` yoki `null` emasmi

Production mode'da bu tekshiruvlar olib tashlanadi (performance overhead'siz).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Database ID — eng yaxshi key:

```tsx
type Article = { id: number; title: string; author: string };

function ArticleList({ articles }: { articles: Article[] }) {
  return (
    <ul>
      {articles.map((article) => (
        <li key={article.id}>
          <h3>{article.title}</h3>
          <p>{article.author}</p>
        </li>
      ))}
    </ul>
  );
}
```

Composite key — agar bitta ID yetarli bo'lmasa:

```tsx
type Enrollment = { studentId: number; courseId: number; grade: string };

function EnrollmentTable({ rows }: { rows: Enrollment[] }) {
  return (
    <table>
      <tbody>
        {rows.map((row) => (
          <tr key={`${row.studentId}_${row.courseId}`}>
            <td>{row.studentId}</td>
            <td>{row.courseId}</td>
            <td>{row.grade}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

UUID generationsi — yangi item yaratilganda **bir marta**, render'da emas:

```tsx
import { useState } from 'react';

type Note = { id: string; text: string };

function NoteApp() {
  const [notes, setNotes] = useState<Note[]>([]);

  const addNote = (text: string) => {
    const newNote: Note = {
      id: crypto.randomUUID(), // ✅ chunki id bir marta generation, keyin saqlanadi
      text,
    };
    setNotes((prev) => [...prev, newNote]);
  };

  return (
    <ul>
      {notes.map((note) => (
        <li key={note.id}>{note.text}</li>
      ))}
    </ul>
  );
}
```

Anti-pattern — render ichida UUID:

```tsx
// ❌ Har render'da yangi key — Reconciler hech qanday item'ni eslatib qolmaydi
function BadNoteList({ notes }: { notes: { text: string }[] }) {
  return (
    <ul>
      {notes.map((note) => (
        <li key={crypto.randomUUID()}>{note.text}</li>
        // Har render'da har item butunlay qayta yaratiladi
        // State, focus, animation — yo'qoladi
      ))}
    </ul>
  );
}
```

Type'ni cheklash — TypeScript aniqroq xato chiqaradi:

```tsx
type ItemProps = { id: number | string; name: string };

function ItemRow({ id, name }: ItemProps) {
  return <li>{name}</li>;
}

function ItemList({ items }: { items: ItemProps[] }) {
  return (
    <ul>
      {items.map((item) => (
        <ItemRow key={item.id} {...item} />
        // ✅ id: number | string — JSX type definition qabul qiladi
      ))}
    </ul>
  );
}
```

</details>

---

## Index `key` Sifatida — Trade-off

### Nazariya

`array.map((item, index) => <li key={index}>...</li>)` — eng oson, lekin eng xavfli pattern. Index `key` sifatida **static list**'larda OK, **dynamic list**'larda esa state bug'ining manbai.

**Index `key` sifatida XAVFSIZ — quyidagi shartlar BIRGA bajarilganda:**

1. List static — qo'shilmaydi, o'chirilmaydi, qayta tartiblanmaydi
2. Item'lar **state'siz** — ichida `useState`, `useRef`, `<input>`, animation yo'q
3. Item'lar **identity'sga muhtoj emas** — item haqida hech narsa "esda saqlanmasin"

Agar uchala shart bajarilsa, index va element positionsi har doim mos keladi va Reconciler hech qachon noto'g'ri matching qilmaydi.

**Index `key` sifatida XAVFLI — quyidagi holatlarda:**

1. Item qo'shilishi yoki o'chirilishi mumkin (CRUD operationlari)
2. Item qayta tartiblanishi mumkin (drag-drop, sort)
3. Item ichida controlled/uncontrolled `<input>`, `<textarea>` bor
4. Item komponentida `useState` bor
5. Item'da animation, focus, scroll position kabi DOM state mavjud

Bu holatlarda index'dan foydalanish bug keltiradi — chunki Reconciler index'ga ko'ra "bu eski item" deb noto'g'ri xulosa qiladi va eski state'ni yangi item'ga "yopishtiradi".

> **`key={index}` haqida runtime warning yo'q:**
> React `key={index}` ishlatishni anti-pattern sifatida hujjatlarda ko'rsatadi, lekin runtime warning chiqarmaydi — chunki ba'zi static holatlarda bu pattern to'g'ri ishlaydi. Bu xulq R16'dan beri o'zgarmagan. Static analysis vositasi sifatida `eslint-plugin-react`'ning `react/no-array-index-key` qoidasi (default'da o'chirilgan, manual'da yoqiladi) bu pattern'ni flag qilishi mumkin.

<details>
<summary><strong>Under the Hood</strong></summary>

Reconciler nuqtai-nazaridan, `key={index}` quyidagicha ishlaydi:

```
Eski list: [A, B, C]
Yangi list: [Z, A, B, C]  (Z prepend qilingan)

Index-based key:
Eski: [A(key=0), B(key=1), C(key=2)]
Yangi: [Z(key=0), A(key=1), B(key=2), C(key=3)]

Reconciler:
  Index 0: eski A (key=0) ↔ yangi Z (key=0) — KEY MOS KELADI
    → A Fiber qayta ishlatiladi, props yangilanadi
    → A komponent state SAQLANADI, lekin u Z'ga aylanadi
  Index 1: eski B ↔ yangi A — bir xil hikoya
  Index 2: eski C ↔ yangi B
  Index 3: yangi C — yangi yaratiladi

Natija: 4 ta state bug — har item state boshqa item'ga ko'chdi
```

Bu sabab — index `key` quyidagilarni **hal qilmaydi**:
- "Yangi item qaysi positionga qo'shilgan?" — javob yo'q
- "Eski va yangi list orasida qaysi item'lar bir xil?" — javob yo'q

Stable ID `key` esa aniq javob beradi:

```
Eski: [A(key='a'), B(key='b'), C(key='c')]
Yangi: [Z(key='z'), A(key='a'), B(key='b'), C(key='c')]

Reconciler:
  key='z' → Map'da yo'q → yangi Fiber Z
  key='a' → Map'da topildi → Fiber A qayta ishlatiladi
  key='b' → Map'da topildi → Fiber B qayta ishlatiladi
  key='c' → Map'da topildi → Fiber C qayta ishlatiladi

Natija: 1 ta yangi DOM insertion, 3 ta state saqlandi
```

Index `key` ishlaydigan context (static) — Reconciler oddiy index-based matching qiladi:

```
Eski: [A, B, C]
Yangi: [A', B', C']  (faqat props o'zgargan, item'lar qaytadan tartiblanmagan)

Index 0: A ↔ A' — props update, state saqlandi
Index 1: B ↔ B'
Index 2: C ↔ C'

Natija: 3 ta props update — to'g'ri
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

✅ **OK — static navigation menyusi:**

```tsx
const NAV_ITEMS = ['Home', 'Products', 'About', 'Contact'] as const;

function Navigation() {
  return (
    <nav>
      <ul>
        {NAV_ITEMS.map((label, index) => (
          <li key={index}>
            <a href={`/${label.toLowerCase()}`}>{label}</a>
          </li>
        ))}
      </ul>
    </nav>
  );
  // ✅ chunki NAV_ITEMS hech qachon o'zgarmaydi, no internal state
}
```

❌ **XATO — todo list (dynamic + state):**

```tsx
import { useState } from 'react';

type Todo = { text: string };

function BadTodoList() {
  const [todos, setTodos] = useState<Todo[]>([
    { text: 'Buy milk' },
    { text: 'Walk dog' },
    { text: 'Read book' },
  ]);

  const removeTodo = (index: number) => {
    setTodos((prev) => prev.filter((_, i) => i !== index));
  };

  return (
    <ul>
      {todos.map((todo, index) => (
        <li key={index}>
          {/* ❌ index key + uncontrolled input + delete = bug */}
          <input defaultValue={todo.text} />
          <button onClick={() => removeTodo(index)}>×</button>
        </li>
      ))}
    </ul>
  );
}

// User scenario (index-key + uncontrolled input bug):
// Initial: todos = [Buy milk(0), Walk dog(1), Read book(2)]
//   Initial DOM inputs (defaultValue'dan):
//     index 0: "Buy milk"
//     index 1: "Walk dog"
//     index 2: "Read book"
//
// 1. User index 1 input'ga "Walk dog (urgent)" yozadi (DOM state, React state'da emas)
// 2. User index 0 (Buy milk) ni o'chiradi → todos = [Walk dog, Read book]
// 3. Reconciliation key={index} bilan:
//    - key=0: eski Buy milk Fiber qayta ishlatildi. DOM input still "Buy milk"
//      (yangi defaultValue="Walk dog" e'tiborga olinmaydi — defaultValue faqat mount'da)
//    - key=1: eski Walk dog Fiber qayta ishlatildi. DOM input still "Walk dog (urgent)"
//      (yangi defaultValue="Read book" e'tiborga olinmaydi)
//    - key=2: eski Read book Fiber unmount qilindi
// 4. Natija: position 1 input'da "Walk dog (urgent)" matni qoldi,
//    lekin uning yonidagi data — Read book. Foydalanuvchining yozgani
//    boshqa item'ga "yopishdi".
// 5. Stable key (todo.id) bo'lsa: Walk dog Fiber sort/shift bilan birga ko'chadi
//    va DOM input value to'g'ri item bilan qoladi.
```

✅ **TO'G'RI — stable ID:**

```tsx
type Todo = { id: string; text: string };

function GoodTodoList() {
  const [todos, setTodos] = useState<Todo[]>([
    { id: 't1', text: 'Buy milk' },
    { id: 't2', text: 'Walk dog' },
    { id: 't3', text: 'Read book' },
  ]);

  const removeTodo = (id: string) => {
    setTodos((prev) => prev.filter((todo) => todo.id !== id));
  };

  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}>
          <input defaultValue={todo.text} />
          <button onClick={() => removeTodo(todo.id)}>×</button>
        </li>
      ))}
    </ul>
  );
  // ✅ chunki todo.id stable — DOM state to'g'ri item bilan qoladi
}
```

✅ **OK — read-only render (no state, no reorder):**

```tsx
type LogEntry = { timestamp: string; message: string };

function LogViewer({ entries }: { entries: LogEntry[] }) {
  return (
    <pre>
      {entries.map((entry, index) => (
        <div key={index}>
          [{entry.timestamp}] {entry.message}
        </div>
      ))}
    </pre>
  );
  // ✅ entries faqat append qilinadi (yangi log)
  // ✅ no state, no reorder, no delete
  // index key bu yerda xavfsiz
}
```

</details>

---

## `key` va Komponent Identity

### Nazariya

React'da `key` o'zgarishi — bu **"bu butunlay boshqa komponent"** signali. `key` o'zgarganda, Reconciler eski Fiber'ni topa olmaydi va yangi Fiber yaratadi:

1. Eski komponent **unmount** qilinadi (cleanup effect'lar ishga tushadi)
2. Yangi komponent **mount** qilinadi (mount effect'lar ishga tushadi)
3. State, ref'lar, DOM — **yangidan** yaratiladi
4. Animation reset bo'ladi

Bu xulq xato emas — `key` o'zgarishi semantik jihatdan "boshqa narsa" degani. Lekin u ikki tomonlama qilich:

- ✅ **Foyda:** Komponent state'ni majburan reset qilish uchun pattern (`33-optimization.md` da batafsil)
- ❌ **Zarar:** Beixtiyor `key` o'zgartirish state'ni yo'qotadi (xato)

**State reset pattern** — `key` ni props'ga bog'lab, state'ni majburan reset qilish:

```tsx
function ProfileForm({ userId }: { userId: number }) {
  return <FormFields key={userId} />;
  // userId o'zgarsa — FormFields qayta mount qilinadi
  // Barcha input state — initial qiymatga qaytadi
}

function FormFields() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  return (
    <>
      <input value={name} onChange={(e) => setName(e.target.value)} />
      <input value={email} onChange={(e) => setEmail(e.target.value)} />
    </>
  );
}
```

Bu pattern alternativasi — `useEffect` ichida `setName('')` qilishdan ko'ra declarative va xatoga moyilroq emas.

**Anti-pattern** — beixtiyor `key` o'zgartirish:

```tsx
// ❌ Math.random() har render — Form har render'da reset bo'ladi
<Form key={Math.random()} />

// ❌ Object reference har render yangi
<Form key={{ user: 'Alice' }} />
```

<details>
<summary><strong>Under the Hood</strong></summary>

Reconciler `key` o'zgarganini ko'radi:

```
Eski Fiber: { type: FormFields, key: 1, stateNode: ... }
Yangi Element: { type: FormFields, key: 2 }

Reconciler:
  oldFiber.key (1) !== newElement.key (2)
  → Fiber match yo'q
  → Eski Fiber unmount queue'ga qo'yiladi (Deletion flag)
  → Yangi Fiber yaratiladi (createFiberFromElement)
```

Commit phase:
1. Mutation phase'da eski Fiber DOM'dan o'chiriladi
2. Cleanup effect'lar ishga tushadi (`useEffect` cleanup, ref detach)
3. Yangi Fiber DOM'ga qo'shiladi
4. Mount effect'lar ishga tushadi
5. Ref'lar attach qilinadi

Bu ish — yangi komponent type bilan bir xil. `type` o'zgarganda ham bir xil hodisa ro'y beradi (cross-ref [`04-reconciliation.md`](04-reconciliation.md) — Type Comparison).

`key` Reconciler uchun **Fiber identity** signali. Type bir xil bo'lib, `key` boshqacha bo'lsa — bu butunlay yangi instance.

```ts
// Internal pseudo-code
function reconcile(oldFiber: Fiber | null, newElement: ReactElement): Fiber {
  if (oldFiber === null) {
    return createFiberFromElement(newElement);
  }

  if (oldFiber.type !== newElement.type) {
    // Type farqli — yangi Fiber
    deleteFiber(oldFiber);
    return createFiberFromElement(newElement);
  }

  if (oldFiber.key !== newElement.key) {
    // Key farqli — yangi Fiber (eski state yo'qoladi)
    deleteFiber(oldFiber);
    return createFiberFromElement(newElement);
  }

  // Type va key bir xil — qayta ishlatish
  return useFiber(oldFiber, newElement.props);
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

State reset pattern — user almashinganda formni reset qilish:

```tsx
import { useState } from 'react';

type User = { id: number; name: string };

function UserEditor({ user }: { user: User }) {
  return (
    <div>
      <h3>Editing: {user.name}</h3>
      <UserForm key={user.id} user={user} />
      {/* user.id o'zgarsa — UserForm qayta mount, state reset */}
    </div>
  );
}

function UserForm({ user }: { user: User }) {
  const [name, setName] = useState(user.name);
  const [draftEmail, setDraftEmail] = useState('');

  return (
    <form>
      <input value={name} onChange={(e) => setName(e.target.value)} />
      <input
        value={draftEmail}
        placeholder="New email"
        onChange={(e) => setDraftEmail(e.target.value)}
      />
    </form>
  );
}

// Foyda:
// 1. User A → User B almashinganda — User A'ning yarim yozilgan email'i yo'qoladi
// 2. Bu — to'g'ri xulq, user input/output tartibida chalkashlik bo'lmaydi
```

Tab almashinganda state reset:

```tsx
import { useState } from 'react';

type Tab = 'profile' | 'settings' | 'billing';

function TabbedEditor() {
  const [activeTab, setActiveTab] = useState<Tab>('profile');

  return (
    <div>
      <nav>
        <button onClick={() => setActiveTab('profile')}>Profile</button>
        <button onClick={() => setActiveTab('settings')}>Settings</button>
        <button onClick={() => setActiveTab('billing')}>Billing</button>
      </nav>
      <TabContent key={activeTab} tab={activeTab} />
      {/* activeTab o'zgarsa — TabContent yangidan mount, state reset */}
    </div>
  );
}

function TabContent({ tab }: { tab: Tab }) {
  const [draft, setDraft] = useState('');
  return (
    <div>
      <h3>{tab}</h3>
      <textarea value={draft} onChange={(e) => setDraft(e.target.value)} />
    </div>
  );
}
```

Anti-pattern — beixtiyor reset:

```tsx
// ❌ Render'da yangi obyekt — har render Form unmount/remount
function BadParent() {
  return <Form key={{ id: 1 }} />;
  // {} har render'da yangi reference — key har doim "yangi"
}

// ❌ Math.random() har render
function BadParent2() {
  return <Form key={Math.random()} />;
}

// ✅ Stable primitive yoki saqlanmasdan
function GoodParent() {
  return <Form />;
  // Yoki agar reset kerak bo'lsa: <Form key={userId} />
}
```

</details>

---

## Nested Lists va Key Scope

### Nazariya

`key` scope **parent'gacha** — ya'ni `key` faqat ushbu parent ichidagi sibling'lar orasida unique bo'lishi shart. Boshqa parent ichidagi `key` qiymatlari bilan bog'lanmaydi.

```tsx
// ✅ OK — har <ul> alohida key scope
<div>
  <ul>
    <li key="1">First UL — A</li>
    <li key="2">First UL — B</li>
  </ul>
  <ul>
    <li key="1">Second UL — A</li>  {/* OK — boshqa parent */}
    <li key="2">Second UL — B</li>
  </ul>
</div>
```

Bu — Reconciler'ning sibling-level matching algorithmidan kelib chiqadi: u faqat **bir parent ichidagi child'lar**ni eshlashtiradi (cross-ref [`04-reconciliation.md`](04-reconciliation.md) — `reconcileChildrenArray`). Boshqa parent — alohida child chain.

**Nested lists** — list ichida list'lar bo'lsa, har map'ning o'z scope'i bor:

```tsx
type Category = {
  id: number;
  name: string;
  products: { id: number; name: string }[];
};

function ProductCatalog({ categories }: { categories: Category[] }) {
  return (
    <div>
      {categories.map((category) => (
        <section key={category.id}>
          <h2>{category.name}</h2>
          <ul>
            {category.products.map((product) => (
              <li key={product.id}>{product.name}</li>
              {/* product.id faqat bu <ul> ichida unique bo'lishi shart */}
              {/* category.id bilan collision yo'q — boshqa parent */}
            ))}
          </ul>
        </section>
      ))}
    </div>
  );
}
```

**Composite key** — agar nested data'da ID'lar global unique emas bo'lsa, parent ID bilan birlashtirilishi mumkin:

```tsx
// product.id faqat category ichida unique
// (lekin har category'da product 1 bo'lishi mumkin)
<li key={`${category.id}_${product.id}`}>...</li>
```

Bu — texnikaga ko'ra **majburiy emas** (har map'ning scope'i alohida), lekin **debugging va clarity** uchun foyda.

<details>
<summary><strong>Under the Hood</strong></summary>

Reconciler nested array'larni traversal paytida har bir level'da alohida key map yaratadi:

```
Level 0: <div>
  Level 1: <section key=1>
    Level 2: <ul>
      Level 3: <li key=A>, <li key=B>, <li key=C>
        ← alohida key map bu sibling'lar orasida
  Level 1: <section key=2>
    Level 2: <ul>
      Level 3: <li key=A>, <li key=B>
        ← yangi key map — Level 1 dan boshqa scope
```

`reconcileChildrenArray` har map'ni birinchi to'liq tugatib, keyin keyingisiga o'tadi (DFS pattern, cross-ref [`03-fiber-architecture.md`](03-fiber-architecture.md) — Tree Traversal). Shu sababli har level alohida memory uchun key map'ga ega.

```ts
// Soddalashtirilgan internal
function reconcileChildren(parent: Fiber, newChildren: ReactElement[]): Fiber[] {
  const existingChildren = mapRemainingChildren(parent.child);
  // existingChildren: Map<key, Fiber> — faqat shu parent uchun
  
  for (const newChild of newChildren) {
    const matchedFiber = existingChildren.get(newChild.key) ?? null;
    // ...
  }
}
```

Bu — har parent uchun alohida `Map<key, Fiber>` — boshqa parent'lardagi key'lar bilan aralashish yo'q.

**`<>` (Fragment) — parent emas:**

```tsx
function FragmentInList() {
  return (
    <>
      <li key="1">A</li>
      <li key="2">B</li>
    </>
  );
}

// Agar bu component <ul> ichida ishlatilsa:
<ul>
  <FragmentInList />
  <li key="3">C</li>
</ul>

// React Fragment'ni "transparent" deb sanaydi — uning child'lari
// to'g'ridan-to'g'ri <ul>'ning child'lari sifatida hisoblanadi
// Demak key=1, key=2, key=3 bir scope'da
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Nested category-product:

```tsx
type Product = { id: number; name: string; price: number };
type Category = { id: number; name: string; products: Product[] };

const catalog: Category[] = [
  {
    id: 1,
    name: 'Electronics',
    products: [
      { id: 101, name: 'Laptop', price: 1200 },
      { id: 102, name: 'Phone', price: 800 },
    ],
  },
  {
    id: 2,
    name: 'Books',
    products: [
      { id: 101, name: 'JS Guide', price: 30 },
      // ↑ id=101 — boshqa category ichida, OK
      { id: 102, name: 'TS Handbook', price: 35 },
    ],
  },
];

function Catalog() {
  return (
    <div>
      {catalog.map((category) => (
        <section key={category.id}>
          <h2>{category.name}</h2>
          <ul>
            {category.products.map((product) => (
              <li key={product.id}>
                {product.name} — ${product.price}
              </li>
            ))}
          </ul>
        </section>
      ))}
    </div>
  );
  // ✅ product.id={101} ikki marta uchraydi — turli <ul> ichida, OK
}
```

Composite key — agar IDs bir global namespace'da bo'lishi kerak (debug/log uchun):

```tsx
function CatalogWithComposite() {
  return (
    <div>
      {catalog.map((category) => (
        <section key={`cat_${category.id}`}>
          <h2>{category.name}</h2>
          <ul>
            {category.products.map((product) => (
              <li key={`cat_${category.id}_prod_${product.id}`}>
                {product.name} — ${product.price}
              </li>
            ))}
          </ul>
        </section>
      ))}
    </div>
  );
}
```

Pivot table — row va column composite:

```tsx
type Cell = { row: number; col: number; value: string };

function PivotTable({ cells }: { cells: Cell[] }) {
  const rows = [...new Set(cells.map((c) => c.row))];
  const cols = [...new Set(cells.map((c) => c.col))];

  return (
    <table>
      <tbody>
        {rows.map((row) => (
          <tr key={row}>
            {cols.map((col) => {
              const cell = cells.find((c) => c.row === row && c.col === col);
              return (
                <td key={`${row}_${col}`}>
                  {cell?.value ?? '-'}
                </td>
              );
            })}
          </tr>
        ))}
      </tbody>
    </table>
  );
  // ✅ Har <tr>'da <td>'lar uchun {row}_{col} composite key
  // Texnik — har <tr> alohida scope, lekin composite key debug'da tushunarli
}
```

</details>

---

## Key Collisions va Composite Keys

### Nazariya

**Key collision** — bir parent ichida ikki yoki ko'p sibling bir xil `key` ga ega bo'lganda yuz beradi. Bu — Reconciler uchun "ikki bir xil identity" degan signal va u quyidagicha ishlov beradi:

1. Birinchi `key` uchun eski Fiber biriktiriladi
2. Ikkinchi va keyingilari — duplikat sifatida flag'lanadi (development warning)
3. Reconciler birinchisini qayta ishlatadi, qolganlari yangidan yaratiladi

Natija — state, focus, animation **birinchi** item'da to'g'ri qoladi, qolganlarda **yo'qoladi**. Bu — "barcha state yo'qoldi" emas, lekin debugging og'ir bo'ladi (chunki bir necha item ishlaydi, bir nechtasi yo'q).

**Common collision sources:**

1. **Database ID'lar har xil source'dan kelganda:** Ikkita API endpoint bir xil `id` qaytarishi mumkin (chunki ID o'z table ichida unique, lekin merge qilinganda collision).

2. **Numeric ID + string ID aralashganda:** `id: 1` va `id: "1"` Reconciler tomonidan **bir xil** sifatida qabul qilinadi (string conversion).

3. **Generatsiya qilingan ID'lar render ichida:** `Math.random()` yoki `crypto.randomUUID()` har render'da yangi qiymat — collision bo'lmasa ham, identity stableity buziladi.

4. **`null` yoki `undefined` ID:** Bir nechta item'da ID bo'lmasa, hammasi `key="undefined"` bo'lib qoladi.

**Yechimlar:**

- **Composite key:** ID'larni birlashtirib, context qo'shing — `${type}_${id}`, `${parentId}_${childId}`
- **Pre-process data:** Server'dan kelgan data'ni client'da normalize qilish — har item'ga unique ID berish
- **UUID at creation:** Yangi item yaratilganda bir marta UUID generation qilib, saqlash

<details>
<summary><strong>Under the Hood</strong></summary>

Reconciler key map quradi:

```ts
function mapRemainingChildren(currentFirstChild: Fiber): Map<string | number, Fiber> {
  const existingChildren = new Map<string | number, Fiber>();
  let existingChild: Fiber | null = currentFirstChild;
  while (existingChild !== null) {
    if (existingChild.key !== null) {
      existingChildren.set(existingChild.key, existingChild);
      // Map.set — agar key bir xil bo'lsa, eskisi overwrite qilinadi
    } else {
      existingChildren.set(existingChild.index, existingChild);
    }
    existingChild = existingChild.sibling;
  }
  return existingChildren;
}
```

`Map.set(key, value)` semantikasiga ko'ra, takroriy `key` eski qiymatni overwrite qiladi. Demak, collision holatida — **oxirgi** item Map'da qoladi (birinchisi emas).

Yangi child'lar walking paytida:

```ts
for (const newChild of newChildren) {
  const matchedFiber = existingChildren.get(newChild.key);
  if (matchedFiber !== undefined) {
    // Match — eski Fiber qayta ishlatiladi
    existingChildren.delete(newChild.key);
    // Bu key bir martadan ko'p ishlatilmaydi
  } else {
    // Yangi Fiber
  }
}
```

Demak, ikki yangi child bir xil `key` bilan kelsa:
- Birinchi yangi child — eski Fiber'ga match qiladi va Map'dan o'chiradi
- Ikkinchi yangi child — Map'da yo'q, **yangidan** Fiber yaratiladi

Bu — "birinchi keladi, birinchi xizmat" pattern. Lekin natija foydalanuvchi nuqtai-nazaridan bug — chunki user "ikki bir xil item" deb hisoblamaydi.

Dev mode'da React reconcilation oldidan `validateChildKeys` orqali har element'da key mavjudligini va duplikatlarni tekshiradi. Duplikat aniqlanganda console warning chiqadi:

```
Warning: Encountered two children with the same key, `someKey`.
Keys should be unique so that components maintain their identity
across updates. Non-unique keys may cause children to be
duplicated and/or omitted — the behavior is unsupported and could
change in a future version.
```

Bu warning faqat development build'da chiqadi (production build'da `__DEV__` flag false, duplikat tekshiruvi olib tashlangan).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Numeric vs string ID collision:

```tsx
type Item = { id: number | string; name: string };

const items: Item[] = [
  { id: 1, name: 'Number ID' },
  { id: '1', name: 'String ID' },
  // Ikki item — id=1 va id="1"
  // React: '' + 1 === '' + '1' → bir xil key!
];

function CollidingList() {
  return (
    <ul>
      {items.map((item) => (
        <li key={item.id}>{item.name}</li>
        // ❌ Console warning: duplicate key "1"
      ))}
    </ul>
  );
}

// ✅ To'g'ri — composite key tip indikatori bilan
function FixedList() {
  return (
    <ul>
      {items.map((item) => (
        <li key={`${typeof item.id}_${item.id}`}>{item.name}</li>
        // key: "number_1" va "string_1" — unique
      ))}
    </ul>
  );
}
```

Multi-source merge — ikki API javobi bir xil `id`:

```tsx
type Notification = { id: number; type: 'message' | 'alert'; text: string };

const messages: Notification[] = [
  { id: 1, type: 'message', text: 'New message' },
  { id: 2, type: 'message', text: 'Friend request' },
];

const alerts: Notification[] = [
  { id: 1, type: 'alert', text: 'Login attempt' },
  // ↑ id=1 messages.id=1 bilan collision
];

const merged = [...messages, ...alerts];

// ✅ Composite key — type + id
function NotificationCenter() {
  return (
    <ul>
      {merged.map((notif) => (
        <li key={`${notif.type}_${notif.id}`}>{notif.text}</li>
      ))}
    </ul>
  );
}
```

`null`/`undefined` ID handling:

```tsx
type Article = { id?: number; title: string };

const articles: Article[] = [
  { id: 1, title: 'First' },
  { title: 'Untitled' }, // id undefined
  { title: 'Another' }, // id undefined
];

// ❌ Xato — ikki item key=undefined → collision
function BadArticles() {
  return (
    <ul>
      {articles.map((article) => (
        <li key={article.id}>{article.title}</li>
      ))}
    </ul>
  );
}

// ✅ Fallback to index'dan ko'ra index bilan composite — chunki list mutate bo'lsa, fallback ham buziladi
// Eng yaxshi yechim: data'ni normalize qilib, har item uchun ID kafolatlash
function FixedArticles() {
  return (
    <ul>
      {articles.map((article, index) => (
        <li key={article.id ?? `auto_${index}`}>{article.title}</li>
        // ⚠️ Bu — vaqtinchalik fallback. Real production'da
        //    data'ni normalize qiling: har item uchun stable ID generation qiling
      ))}
    </ul>
  );
}

// ✅✅ Eng yaxshi — data'ni client'da normalize qilish
import { useMemo } from 'react';

function NormalizedArticles({ raw }: { raw: Article[] }) {
  const normalized = useMemo(
    () =>
      raw.map((article, index) => ({
        ...article,
        id: article.id ?? -(index + 1), // negative id — auto-generated
      })),
    [raw]
  );

  return (
    <ul>
      {normalized.map((article) => (
        <li key={article.id}>{article.title}</li>
      ))}
    </ul>
  );
}
```

</details>

---

## Reordering Performance — Reconciler Bog'lanishi

### Nazariya

`key` ning eng katta foydasi — **reorder** (qayta tartiblash) operationsi. Drag-and-drop, sort, filter — barcha holatda item positionsi o'zgaradi, lekin item identity saqlanadi.

Stable `key` bilan, Reconciler `lastPlacedIndex` greedy algorithmi orqali kerakli DOM operation soni'ni minimizatsiya qiladi:

- **Faqat move qilingan item'lar** uchun `insertBefore` chaqiriladi
- **Ko'pchilik item'lar** joyida qoladi (state, DOM saqlanadi)
- **Yangi item'lar** uchun `appendChild`/`insertBefore`
- **O'chirilgan item'lar** uchun `removeChild`

`key` bo'lmasa, **har item** index positionsiga ko'ra qayta yangilanadi:

- Har item'ning props'i yangilanadi
- DOM'da har item'ning innerText/attribute'lari qayta yoziladi
- Komponent state'i positionga "yopishadi", item identity'ga emas

10000 element'lik list'da reverse operation:

```
Stable key bilan:
  10000 ta DOM move (insertBefore)
  0 ta props update
  0 ta state loss

Index key bilan:
  0 ta DOM move
  10000 ta props update
  10000 ta state loss (har item state boshqasiga ko'chdi)
```

Stable `key` ning asosiy foydasi — **state preservation** va **kerakli DOM operationlari soni**ning kamayishi. Index key holatida React har item'ni yangi item deb sanaydi va props/text/attribute'larni qayta yozadi; stable key holatida React faqat haqiqatan ham o'zgargan item'larni yangilaydi va boshqalarini joyida qoldiradi. DOM `insertBefore` reflow trigger qiladi, lekin u props/attribute update + state reset birikmasidan kichik xarajat.

> **Performance:** Konkret benchmark raqamlari contextga (browser, item murakkabligi, item soni) qattiq bog'liq. Praktikada — yirik list'larda (10k+) reorder uchun stable key MAJBURIY. Kichik list'larda (100-element gacha) performance farqi sezilmasligi mumkin, lekin state correctness sababli stable key har doim tavsiya qilinadi.

<details>
<summary><strong>Under the Hood</strong></summary>

`reconcileChildrenArray` ikki fazada ishlaydi (cross-ref [`04-reconciliation.md`](04-reconciliation.md) — `lastPlacedIndex` move detection batafsil yoritilgan):

**Faza 1 — Sequential walk:**

Yangi va eski child'lar position bo'yicha solishtiriladi, key mos kelmagan birinchi positionga yetguncha. Agar oxirigacha mos kelsa — minimal ish (props update).

**Faza 2 — Key map:**

Mos kelmagan positiondan boshlab, qolgan eski child'lar `Map<key, Fiber>` ga to'planadi. Yangi child'lar bu Map'dan key orqali izlanadi:

```ts
let lastPlacedIndex = 0;
let newIndex = 0;

for (const newChild of remainingNewChildren) {
  const matchedFiber = existingMap.get(newChild.key);
  
  if (matchedFiber !== undefined) {
    const oldIndex = matchedFiber.index;
    
    if (oldIndex < lastPlacedIndex) {
      // Move kerak — newChild oldindagi item'dan keyinroq edi
      matchedFiber.flags |= Placement;
    } else {
      // No move — lastPlacedIndex yangilanadi
      lastPlacedIndex = oldIndex;
    }
    
    existingMap.delete(newChild.key);
  } else {
    // Yangi Fiber
    const fiber = createFiberFromElement(newChild);
    fiber.flags |= Placement;
  }
  
  newIndex++;
}

// existingMap'da qolgan barcha Fiber'lar — o'chirilishi kerak
for (const oldFiber of existingMap.values()) {
  oldFiber.flags |= Deletion;
}
```

`Placement` flag — Commit Phase'da DOM operationsiga aylanadi:

- Yangi Fiber + Placement — `appendChild` yoki `insertBefore`
- Eski Fiber + Placement — `insertBefore` (boshqa positionga ko'chirish)
- Eski Fiber + Deletion — `removeChild`

**Greedy algorithm trade-off:**

Bu O(n) algorithm, lekin **optimal emas**. Misol:

```
Eski: [A, B, C, D, E]
Yangi: [E, A, B, C, D]  (E boshga ko'chdi)
```

Optimal yechim: 1 ta move (E'ni boshga ko'chirish).
Greedy yechim: 4 ta move (A, B, C, D — chunki E ulardan oldin keldi).

Reverse-friendly **emas** — chunki algorithm "left to right" walk qiladi va `lastPlacedIndex` monoton ortib boradi (kamaymaydi, lekin tenglik mumkin).

LCS (Longest Common Subsequence) algorithmi optimal yechimni topadi (1 move bu misolda), lekin O(n²) kompleksiti — yirik list'larda real sekinroq. React O(n) greedy'ni tanladi: ko'pchilik praktik holatlarda yetarli, va hech qachon hatto eng yomon holatda ham `n` move'dan ortmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Sortable list — drag-drop:

```tsx
import { useState } from 'react';

type Task = { id: string; title: string; order: number };

function SortableTaskList({ initialTasks }: { initialTasks: Task[] }) {
  const [tasks, setTasks] = useState(initialTasks);

  const moveUp = (id: string) => {
    setTasks((prev) => {
      const index = prev.findIndex((t) => t.id === id);
      if (index <= 0) return prev;
      const next = [...prev];
      [next[index - 1], next[index]] = [next[index], next[index - 1]];
      return next;
    });
  };

  return (
    <ul>
      {tasks.map((task) => (
        <li key={task.id}>
          {/* ✅ stable key — Reconciler move detect qiladi */}
          {/* DOM move yuz beradi, lekin task'ning input state saqlanadi */}
          <input defaultValue={task.title} />
          <button onClick={() => moveUp(task.id)}>↑</button>
        </li>
      ))}
    </ul>
  );
}
```

Sort by column — table:

```tsx
type Row = { id: number; name: string; age: number };

function SortableTable({ rows }: { rows: Row[] }) {
  const [sortBy, setSortBy] = useState<'name' | 'age'>('name');

  const sorted = [...rows].sort((a, b) => {
    if (sortBy === 'name') return a.name.localeCompare(b.name);
    return a.age - b.age;
  });

  return (
    <div>
      <button onClick={() => setSortBy('name')}>Sort by name</button>
      <button onClick={() => setSortBy('age')}>Sort by age</button>
      <table>
        <tbody>
          {sorted.map((row) => (
            <tr key={row.id}>
              <td>{row.name}</td>
              <td>{row.age}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
    // ✅ row.id stable — sort har <tr>'ni move qiladi, props ham state ham to'g'ri qoladi
  );
}
```

Filter + render — animation friendly:

```tsx
import { useState } from 'react';

type Product = { id: number; name: string; category: string };

function FilteredCatalog({ products }: { products: Product[] }) {
  const [filter, setFilter] = useState('');

  const filtered = products.filter((p) =>
    p.name.toLowerCase().includes(filter.toLowerCase())
  );

  return (
    <div>
      <input
        value={filter}
        onChange={(e) => setFilter(e.target.value)}
        placeholder="Search..."
      />
      <ul>
        {filtered.map((product) => (
          <li key={product.id} className="fade-in">
            {product.name}
          </li>
        ))}
      </ul>
    </div>
  );
  // ✅ Reconciler matched item'larni saqlaydi, faqat yangi/o'chirilgan item'larda animation ishga tushadi
  // ❌ Agar key={index} bo'lsa, har filter o'zgarganda barcha item qayta render
}
```

</details>

---

## Fragment va Special Cases

### Nazariya

**`<>...</>` (short Fragment)** — `key` qabul **qila olmaydi**. Bu — JSX transform cheklovi. Agar Fragment'da `key` kerak bo'lsa, **explicit Fragment** ishlatiladi:

```tsx
import { Fragment } from 'react';

// ❌ Short syntax key qo'llamaydi
<>
  <li>A</li>
  <li>B</li>
</>

// ✅ Explicit Fragment + key
<Fragment key={group.id}>
  <li>A</li>
  <li>B</li>
</Fragment>
```

Bu — list ichida ikki yoki ko'p element'ni guruhlangan holda render qilish kerak bo'lganda muhim:

```tsx
type Group = { id: number; items: { id: number; name: string }[] };

function GroupedItems({ groups }: { groups: Group[] }) {
  return (
    <ul>
      {groups.map((group) => (
        <Fragment key={group.id}>
          <li className="header">{group.id}</li>
          {group.items.map((item) => (
            <li key={item.id}>{item.name}</li>
          ))}
        </Fragment>
      ))}
    </ul>
  );
}
```

**Empty list** — JSX engine `[]`, `null`, `undefined`, `false` ni "render qilma" deb biladi:

```tsx
function EmptyAwareList({ items }: { items: string[] }) {
  if (items.length === 0) {
    return <p>No items</p>;
  }
  return (
    <ul>
      {items.map((item, i) => (
        <li key={i}>{item}</li>
      ))}
    </ul>
  );
}
```

`items.length && <ul>...</ul>` pattern — **0 trap** xavfi bor (cross-ref [`07-jsx.md`](07-jsx.md) — Conditional rendering 0 trap):

```tsx
// ❌ 0 trap — items.length=0 bo'lsa "0" render qilinadi (li tag emas, balki "0" matni)
{items.length && <ul>{items.map(...)}</ul>}

// ✅ Boolean ga o'tkazish yoki ternary
{items.length > 0 && <ul>{items.map(...)}</ul>}
{items.length > 0 ? <ul>{items.map(...)}</ul> : <p>Empty</p>}
```

**Streaming va Suspense** — list ichida async data yuklash uchun:

```tsx
import { Suspense } from 'react';

function ProductList() {
  return (
    <ul>
      {productIds.map((id) => (
        <Suspense key={id} fallback={<li>Loading {id}...</li>}>
          <ProductRow id={id} />
        </Suspense>
      ))}
    </ul>
  );
  // Har <Suspense> mustaqil hydrate bo'ladi (cross-ref 06-hydration.md)
}
```

**Virtualization** — yirik list'larda faqat ko'rinadigan item'larni render qilish texnikasi. Pure React'da implement qilish mumkin (preview mashqlarda), kutubxona — `react-window`, `@tanstack/react-virtual`. To'liq kursning [`36-virtualization.md`](36-virtualization.md) bo'limi mavzusi.

<details>
<summary><strong>Under the Hood</strong></summary>

JSX transform `<>...</>` ni quyidagicha o'zgartiradi (R17+ automatic runtime):

```tsx
// JSX
<>
  <li>A</li>
  <li>B</li>
</>

// Transform natijasi
import { Fragment as _Fragment, jsx as _jsx, jsxs as _jsxs } from 'react/jsx-runtime';

_jsxs(_Fragment, {
  children: [_jsx('li', { children: 'A' }), _jsx('li', { children: 'B' })],
});
// key argument YO'Q — short syntax JSX parser darajasida uni qo'llab-quvvatlamaydi
```

Explicit `<Fragment key="...">`:

```tsx
import { Fragment } from 'react';

<Fragment key="g1">
  <li>A</li>
  <li>B</li>
</Fragment>

// Transform
_jsxs(Fragment, {
  children: [_jsx('li', { children: 'A' }), _jsx('li', { children: 'B' })],
}, 'g1');
// ↑ key — 3-argument
```

`Fragment` — `react` paketning Symbol export'i (`Symbol.for('react.fragment')`). Reconciler bu Symbol'ni ko'rib, "transparent host" deb hisoblaydi — uning child'lari to'g'ridan-to'g'ri parent'ga ulanadi (DOM'da Fragment node yo'q).

`<Suspense key={...}>` — boundary'ning identity'sini belgilaydi. Key o'zgarsa, eski Suspense unmount qilinadi va yangi mount — ya'ni state, fallback timer, va uchirilgan promise'lar reset qilinadi.

Empty array `[]` — JSX engine quyidagicha ishlov beradi:

```tsx
{[]}
// _jsx ga child sifatida bo'sh array uzatiladi
// Reconciler: array — yetishtirilmagan bola yo'q
// DOM'ga hech narsa qo'shilmaydi
```

`null`, `undefined`, `false` ham shu kabi — Reconciler ularni "render qilma" sifatida tushunadi.

`true` esa **render qilinmaydi** va xato hosil bo'lmaydi (lekin debug paytida silent xulq sezilmasdan o'tib ketishi mumkin):

```tsx
{condition && true}  // Hech narsa render qilinmaydi (true)
{condition && 'text'}  // 'text' render qilinadi
{condition && 0}  // '0' render qilinadi (0 trap!)
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Description list — `<dt>` va `<dd>` juftliklar:

```tsx
import { Fragment } from 'react';

type Definition = { id: number; term: string; definition: string };

function Glossary({ entries }: { entries: Definition[] }) {
  return (
    <dl>
      {entries.map((entry) => (
        <Fragment key={entry.id}>
          <dt>{entry.term}</dt>
          <dd>{entry.definition}</dd>
        </Fragment>
        // ✅ Fragment ikki element'ni bitta logik group qiladi
        // <dl> ichida <dt> va <dd> ketma-ket joylashadi
      ))}
    </dl>
  );
}
```

Suspense per item — har item alohida loading:

```tsx
import { Suspense } from 'react';

type CommentProps = { id: number };

function CommentRow({ id }: CommentProps) {
  // use(fetchComment(id)) — R19 use API (cross-ref 23-r19-hooks.md)
  return <li>Comment #{id}</li>;
}

function CommentList({ ids }: { ids: number[] }) {
  return (
    <ul>
      {ids.map((id) => (
        <Suspense key={id} fallback={<li>Loading...</li>}>
          <CommentRow id={id} />
        </Suspense>
      ))}
    </ul>
  );
  // Har comment mustaqil yuklanadi va hydrate bo'ladi
}
```

Empty list patterns:

```tsx
type Article = { id: number; title: string };

function ArticleListWithEmpty({ articles }: { articles: Article[] }) {
  if (articles.length === 0) {
    return <p className="empty">No articles published yet.</p>;
  }

  return (
    <ul>
      {articles.map((article) => (
        <li key={article.id}>{article.title}</li>
      ))}
    </ul>
  );
}

// Ternary pattern
function ArticleListTernary({ articles }: { articles: Article[] }) {
  return articles.length > 0 ? (
    <ul>
      {articles.map((a) => (
        <li key={a.id}>{a.title}</li>
      ))}
    </ul>
  ) : (
    <p>No articles</p>
  );
}

// Bitta UL ichida fallback
function ArticleListWithFallback({ articles }: { articles: Article[] }) {
  return (
    <ul>
      {articles.length === 0 && <li className="empty">No articles</li>}
      {articles.map((a) => (
        <li key={a.id}>{a.title}</li>
      ))}
    </ul>
  );
}
```

Virtualization preview — windowing sodda usul:

```tsx
import { useState, useMemo } from 'react';

type Item = { id: number; text: string };

function VirtualizedList({ items }: { items: Item[] }) {
  const [scrollTop, setScrollTop] = useState(0);
  const ITEM_HEIGHT = 30;
  const VISIBLE_COUNT = 20;

  const startIndex = Math.floor(scrollTop / ITEM_HEIGHT);
  const visible = useMemo(
    () => items.slice(startIndex, startIndex + VISIBLE_COUNT),
    [items, startIndex]
  );

  return (
    <div
      style={{ height: 600, overflowY: 'auto' }}
      onScroll={(e) => setScrollTop(e.currentTarget.scrollTop)}
    >
      <div style={{ height: items.length * ITEM_HEIGHT, position: 'relative' }}>
        {visible.map((item, i) => (
          <div
            key={item.id}
            style={{
              position: 'absolute',
              top: (startIndex + i) * ITEM_HEIGHT,
              height: ITEM_HEIGHT,
            }}
          >
            {item.text}
          </div>
        ))}
      </div>
    </div>
  );
  // To'liq virtualization — react-window yoki @tanstack/react-virtual ishlatish tavsiya qilinadi
  // Bu sodda misol shunchaki window concept'ni tushuntiradi
}
```

</details>

---

## Edge Cases va Gotchas

### Gotcha 1: Spread bilan `key` — `_jsx` runtime'da spread g'olib

`key` — JSX-level maxsus token: parser/transform explicit `key={...}` attribute'ini `_jsx(type, props, key)` chaqirig'idagi alohida (uchinchi) argument sifatida uzatadi. Lekin `{...item}` spread qilinganida `item.key` field config object ichiga tushadi va `_jsx` runtime'ning ichki logikasi config.key'ni 3-argumentdan **keyin** tekshirib, uni **override qiladi**:

```tsx
const item = { id: 1, name: 'Item', key: 'spread-key' };

<Item {...item} key={item.id} />
// JSX transform (automatic): _jsx(Item, { ...item }, item.id)
// config = { id: 1, name: 'Item', key: 'spread-key' }
// maybeKey = item.id (1)
//
// _jsx runtime ichki kodi (React source 18.x/19):
//   let key = null;
//   if (maybeKey !== undefined) key = '' + maybeKey;       // key = "1"
//   if (hasValidKey(config))    key = '' + config.key;     // key = "spread-key" (OVERRIDE!)
//
// Natija: element.key = "spread-key" — spread g'olib, explicit `key={item.id}` qoldi
```

Bu — confusing bug manbai: foydalanuvchi `<Item key={item.id} />` deb yozayotganda, spread orqali kelgan `key` field uni override qiladi. React source code'idagi izoh ham bu muammoni tan oladi va `key` spread'ni deprecate qilish rejasi bor (R18.3+/R19'da dev warning chiqadi):

```
Warning: A props object containing a "key" prop is being spread into JSX:
  let props = { key: someKey, ...otherProps };
  <ComponentName {...props} />
React keys must be passed directly to JSX without using spread:
  let props = otherProps;
  <ComponentName key={someKey} {...props} />
```

**Yechim — spread source'idan `key` ni olib tashlash** (bu warning'ning rasmiy "fix" pattern'i ham):

```tsx
// ✅ Spread'dan key'ni olib tashlash
const { key: _, ...rest } = item;
<Item key={item.id} {...rest} />
// Endi config'da key yo'q, maybeKey g'olib
```

Object'larda `key` nomli field saqlanishidan saqlanish ham yaxshi pattern — ayniqsa server'dan keladigan obyekt'larda `key` reserved nom sifatida ishlatmaslik tavsiya etiladi.

---

### Gotcha 2: Array mutation bilan re-render bo'lmaslik

```tsx
const [items, setItems] = useState<Item[]>([...]);

// ❌ Array'ni in-place mutate qilish — re-render bo'lmaydi
const removeFirst = () => {
  items.shift();  // Array reference bir xil
  setItems(items); // Object.is(items, items) === true → bailout
};

// ✅ To'g'ri — yangi array
const removeFirst = () => {
  setItems((prev) => prev.slice(1));
};
```

Bu — `key` bilan bog'liq bo'lmagan, lekin list rendering contextida ko'p uchraydigan xato. React `Object.is` bilan state taqqoslaydi (cross-ref [`04-reconciliation.md`](04-reconciliation.md) — Bailout).

---

### Gotcha 3: Same `key` Two Different Sibling List'larda

```tsx
function SplitList({ items }: { items: Item[] }) {
  const [active, inactive] = items.reduce<[Item[], Item[]]>(
    ([a, i], item) => (item.active ? [[...a, item], i] : [a, [...i, item]]),
    [[], []]
  );

  return (
    <>
      <ul className="active">
        {active.map((i) => <li key={i.id}>{i.name}</li>)}
      </ul>
      <ul className="inactive">
        {inactive.map((i) => <li key={i.id}>{i.name}</li>)}
      </ul>
    </>
  );
}

// Item active=true → false bo'lsa:
// `active` list'dan o'chiriladi (unmount)
// `inactive` list'ga qo'shiladi (mount)
// Item o'zgaruvchan state'i — YO'QOLADI

// Bu — `key` scope sabab. Active va inactive ikki alohida parent → key bog'lanmaydi
// Item bir <ul>'dan boshqasiga "ko'chirilgan" emas, balki birida unmount, boshqasida mount
```

**Yechim:** Agar item state'i kerak bo'lsa — bir parent ichida `display: none` orqali yashirish, yoki state'ni parent'ga lift qilish.

---

### Gotcha 4: Conditional siblings va `key` aniqligi

```tsx
{showHeader && <Header key="hdr" />}
{items.map((i) => <Item key={i.id} {...i} />)}
```

`showHeader` true → false bo'lganda, Header unmount qilinadi. Item'lar positionsi 1 ga siljimaydi (Reconciler key bilan eshlashtiradi). Lekin agar key'siz bo'lsa — index positionsi siljiydi va xato yuz beradi.

Conditional element'larda har doim explicit `key` qo'shing.

---

### Gotcha 5: Sort + `index` `key` da'vosi

```tsx
const sorted = [...items].sort((a, b) => a.name.localeCompare(b.name));

return (
  <ul>
    {sorted.map((item, index) => (
      <li key={index}>{item.name}</li>
      // ❌ Sort qilingan, lekin index key — har sort'da har item butunlay yangi
    ))}
  </ul>
);
```

Sort har item'ning index'ini o'zgartiradi. `key={index}` ishlatilsa, Reconciler **hech qanday match topa olmaydi** va barcha item'lar qayta yaratiladi (state, focus, animation — yo'qoladi).

**Yechim:** `key={item.id}` — sort qilgandan keyin ham item identity stable.

---

## Common Mistakes

### ❌ Xato 1: Index `key` Dynamic List'da

```tsx
// ❌ Anti-pattern
function TodoList({ todos }: { todos: Todo[] }) {
  return (
    <ul>
      {todos.map((todo, index) => (
        <li key={index}>
          <input defaultValue={todo.text} />
        </li>
      ))}
    </ul>
  );
}

// ✅ To'g'ri
function TodoList({ todos }: { todos: Todo[] }) {
  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}>
          <input defaultValue={todo.text} />
        </li>
      ))}
    </ul>
  );
}
```

**Sabab:** `index` qator positionsi — item kelib chiqishi emas. Item qo'shilsa/o'chirilsa/sort qilinsa, index'lar siljiydi va Reconciler eski state'ni noto'g'ri item'ga "yopishtiradi".

---

### ❌ Xato 2: `Math.random()` yoki Render-time UUID `key` da

```tsx
// ❌ Anti-pattern — har render'da yangi key
function ItemList({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map((item) => (
        <li key={Math.random()}>{item.name}</li>
      ))}
    </ul>
  );
}

// ❌ Yana anti-pattern — render ichida UUID
function ItemList({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map((item) => (
        <li key={crypto.randomUUID()}>{item.name}</li>
      ))}
    </ul>
  );
}

// ✅ To'g'ri — UUID item yaratilganda bir marta
function App() {
  const [items, setItems] = useState<Item[]>([]);
  const addItem = (name: string) =>
    setItems((prev) => [...prev, { id: crypto.randomUUID(), name }]);

  return (
    <ul>
      {items.map((item) => <li key={item.id}>{item.name}</li>)}
    </ul>
  );
}
```

**Sabab:** `key` har render'da yangi bo'lsa, Reconciler hech qachon "bu eski item" deb topa olmaydi — har item butunlay qayta yaratiladi (state, DOM, animation reset).

---

### ❌ Xato 3: `key` Ni Komponent Ichida Ishlatishga Urinish

```tsx
// ❌ Anti-pattern — key prop sifatida ishlatish
type ItemProps = { name: string };

function Item({ name, key }: ItemProps & { key: string }) {
  // ❌ key bu yerda accessible EMAS
  console.log(key); // undefined
  return <li>{name}</li>;
}

// ✅ To'g'ri — kerakli value'ni alohida prop bilan
type ItemProps = { id: string; name: string };

function Item({ id, name }: ItemProps) {
  return <li data-id={id}>{name}</li>;
}

function ItemList({ items }: { items: ItemProps[] }) {
  return (
    <ul>
      {items.map((item) => (
        <Item key={item.id} id={item.id} name={item.name} />
      ))}
    </ul>
  );
}
```

**Sabab:** `key` JSX transform tomonidan props'dan ajratib olinadi va Reconciler uchun internal slot'ga saqlanadi. Komponent function'iga prop sifatida uzatilmaydi.

---

### ❌ Xato 4: Fragment'da `key` Qo'ymaslik

```tsx
type Group = { id: number; items: Item[] };

// ❌ Anti-pattern — short Fragment + nested map = key yo'q
function GroupedList({ groups }: { groups: Group[] }) {
  return (
    <ul>
      {groups.map((group) => (
        <>
          {/* ❌ <> key qabul qilmaydi */}
          <li>{group.id}</li>
          {group.items.map((item) => (
            <li key={item.id}>{item.name}</li>
          ))}
        </>
      ))}
    </ul>
  );
}

// ✅ To'g'ri — explicit Fragment
import { Fragment } from 'react';

function GroupedList({ groups }: { groups: Group[] }) {
  return (
    <ul>
      {groups.map((group) => (
        <Fragment key={group.id}>
          <li>{group.id}</li>
          {group.items.map((item) => (
            <li key={item.id}>{item.name}</li>
          ))}
        </Fragment>
      ))}
    </ul>
  );
}
```

**Sabab:** `<>...</>` short syntax `key` propni qo'llab-quvvatlamaydi. Group'larni list ichida render qilish uchun explicit `<Fragment key={...}>` kerak.

---

### ❌ Xato 5: Object Reference `key` Sifatida

```tsx
type Item = { name: string; data: object };

// ❌ Anti-pattern — object key
function BadList({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map((item) => (
        <li key={item}>{item.name}</li>
        // String conversion: '[object Object]' — barcha item'lar bir xil key!
      ))}
    </ul>
  );
}

// ❌ Yana xato — object reference
function BadList2({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map((item) => (
        <li key={item.data}>{item.name}</li>
        // Yana '[object Object]'
      ))}
    </ul>
  );
}

// ✅ To'g'ri — string yoki number primitive
type Item = { id: number; name: string };

function GoodList({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map((item) => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  );
}
```

**Sabab:** React `key` qiymatini stringga conversion qiladi (`'' + key`). Object'ning default `toString()` natijasi `'[object Object]'` — barcha object'lar uchun bir xil. Natijada barcha item'lar duplikat key bilan qoladi.

---

## Amaliy Mashqlar

### Mashq 1: Sodda Product List (Oson)

`Product` array'idan `<ul>` render qiling. Har item — `<li>` ichida nom va narx. To'g'ri `key` ishlating.

```tsx
type Product = { id: number; name: string; price: number };

const products: Product[] = [
  { id: 1, name: 'Keyboard', price: 49 },
  { id: 2, name: 'Mouse', price: 19 },
  { id: 3, name: 'Monitor', price: 299 },
];

function ProductList() {
  // TODO: <ul> ichida har product uchun <li>
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function ProductList() {
  return (
    <ul>
      {products.map((product) => (
        <li key={product.id}>
          {product.name} — ${product.price}
        </li>
      ))}
    </ul>
  );
}
```

`product.id` stable database ID — eng yaxshi key tanlovi. `index` ishlatish OK bo'lardi (static list), lekin amaliyotda har doim ID afzal — chunki list o'sib borishi mumkin.

</details>

---

### Mashq 2: Nested Category-Product Render (Oson)

Quyidagi nested data'ni render qiling. Har category — `<section>`, ichida `<h2>` va `<ul>` bilan product'lar. Har element to'g'ri `key`'ga ega bo'lsin.

```tsx
type Product = { id: number; name: string };
type Category = { id: number; name: string; products: Product[] };

const data: Category[] = [
  {
    id: 1,
    name: 'Electronics',
    products: [
      { id: 1, name: 'Phone' },
      { id: 2, name: 'Laptop' },
    ],
  },
  {
    id: 2,
    name: 'Books',
    products: [
      { id: 1, name: 'JavaScript Guide' },
      // ↑ id=1 — boshqa <ul> ichida, OK
    ],
  },
];
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
function Catalog() {
  return (
    <div>
      {data.map((category) => (
        <section key={category.id}>
          <h2>{category.name}</h2>
          <ul>
            {category.products.map((product) => (
              <li key={product.id}>{product.name}</li>
            ))}
          </ul>
        </section>
      ))}
    </div>
  );
}
```

`category.id` va `product.id` har xil scope'da — collision yo'q. Har `<ul>` o'z key map'iga ega.

</details>

---

### Mashq 3: Sortable Todo Saqlash (O'rta)

Todo list'ni sort qilganda, har todo'ning `<input>` value'si saqlanishi kerak. Quyidagi komponent xato — bug'ni toping va tuzating.

```tsx
import { useState } from 'react';

type Todo = { id: number; text: string };

function SortableTodos() {
  const [todos] = useState<Todo[]>([
    { id: 1, text: 'Buy milk' },
    { id: 2, text: 'Walk dog' },
    { id: 3, text: 'Read book' },
  ]);
  const [sortAsc, setSortAsc] = useState(true);

  const sorted = [...todos].sort((a, b) =>
    sortAsc ? a.text.localeCompare(b.text) : b.text.localeCompare(a.text)
  );

  return (
    <div>
      <button onClick={() => setSortAsc((s) => !s)}>
        Sort {sortAsc ? '↑' : '↓'}
      </button>
      <ul>
        {sorted.map((todo, index) => (
          <li key={index}>
            <input defaultValue={todo.text} />
          </li>
        ))}
      </ul>
    </div>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

Bug: `key={index}`. Sort har item'ning index'ini o'zgartiradi va Reconciler eski item'ni topa olmaydi — barcha `<input>` qayta yaratiladi va user yozgani yo'qoladi.

```tsx
function SortableTodos() {
  const [todos] = useState<Todo[]>([
    { id: 1, text: 'Buy milk' },
    { id: 2, text: 'Walk dog' },
    { id: 3, text: 'Read book' },
  ]);
  const [sortAsc, setSortAsc] = useState(true);

  const sorted = [...todos].sort((a, b) =>
    sortAsc ? a.text.localeCompare(b.text) : b.text.localeCompare(a.text)
  );

  return (
    <div>
      <button onClick={() => setSortAsc((s) => !s)}>
        Sort {sortAsc ? '↑' : '↓'}
      </button>
      <ul>
        {sorted.map((todo) => (
          <li key={todo.id}>
            {/* ✅ todo.id stable — sort'da DOM move qiladi, input value saqlanadi */}
            <input defaultValue={todo.text} />
          </li>
        ))}
      </ul>
    </div>
  );
}
```

`key={todo.id}` orqali Reconciler har item'ni stable identity bilan eshlashtiradi. Sort natijasida DOM move yuz beradi (`insertBefore`), lekin har `<input>` o'zining state'i bilan birga ko'chadi.

</details>

---

### Mashq 4: Filter va Reset State (O'rta)

`UserCard` komponenti — `useState` ichida draft message saqlaydi. User filter qilganda, **boshqa user'larning** draft'i yo'qolmasligi kerak. Quyidagi xato kodni tuzating.

```tsx
import { useState } from 'react';

type User = { id: number; name: string };

function UserCard({ user }: { user: User }) {
  const [draft, setDraft] = useState('');
  return (
    <div>
      <h3>{user.name}</h3>
      <textarea
        value={draft}
        onChange={(e) => setDraft(e.target.value)}
        placeholder="Draft message..."
      />
    </div>
  );
}

function UserList({ users }: { users: User[] }) {
  const [filter, setFilter] = useState('');
  const filtered = users.filter((u) =>
    u.name.toLowerCase().includes(filter.toLowerCase())
  );

  return (
    <div>
      <input value={filter} onChange={(e) => setFilter(e.target.value)} />
      {filtered.map((user, index) => (
        <UserCard key={index} user={user} />
      ))}
    </div>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

Bug: `key={index}` filter natijasida indeks positionsi o'zgaradi, va `<UserCard>`'lar boshqa user'ning draft'ini ko'rsatadi.

```tsx
function UserList({ users }: { users: User[] }) {
  const [filter, setFilter] = useState('');
  const filtered = users.filter((u) =>
    u.name.toLowerCase().includes(filter.toLowerCase())
  );

  return (
    <div>
      <input value={filter} onChange={(e) => setFilter(e.target.value)} />
      {filtered.map((user) => (
        <UserCard key={user.id} user={user} />
        //         ↑ user.id — stable identity
      ))}
    </div>
  );
}
```

Endi filter natijasida React har card'ni `user.id` orqali eshlashtiradi:
- Filter'da qolgan user'lar — eski Fiber qayta ishlatiladi, draft saqlanadi
- Filter'dan o'chgan user'lar — unmount, draft yo'qoladi (kutilgan)
- Filter'ga qaytgan user — agar avval unmount qilingan bo'lsa, yangi mount + bo'sh draft (kutilgan)

</details>

---

### Mashq 5: Virtualized List Bilan State Saqlash (Qiyin)

10000 element'lik list'ni virtualization qiling. Faqat ko'rinadigan 20 element render qilinadi. **Lekin** har item ichida `<input>` bor va user input qiymati scroll'ga qaramay saqlanishi kerak.

```tsx
type Item = { id: number; label: string };

const items: Item[] = Array.from({ length: 10000 }, (_, i) => ({
  id: i,
  label: `Item ${i}`,
}));

function VirtualList() {
  // TODO: scrollTop'ga qarab visible window
  // TODO: per-item input value'larni saqlash
}
```

<details>
<summary><strong>Javob</strong></summary>

Virtualization input value'larni komponent ichida saqlasa, scroll bilan unmount/remount yuz beradi va value yo'qoladi. Yechim — value'larni parent state'da saqlash.

```tsx
import { useState, useMemo, useCallback } from 'react';

type Item = { id: number; label: string };

const items: Item[] = Array.from({ length: 10000 }, (_, i) => ({
  id: i,
  label: `Item ${i}`,
}));

const ITEM_HEIGHT = 30;
const VISIBLE_COUNT = 20;
const CONTAINER_HEIGHT = 600;

function VirtualList() {
  const [scrollTop, setScrollTop] = useState(0);
  const [values, setValues] = useState<Map<number, string>>(new Map());

  const startIndex = Math.max(
    0,
    Math.floor(scrollTop / ITEM_HEIGHT) - 2
  );
  const endIndex = Math.min(
    items.length,
    startIndex + VISIBLE_COUNT + 4
  );

  const visible = useMemo(
    () => items.slice(startIndex, endIndex),
    [startIndex, endIndex]
  );

  const handleChange = useCallback((id: number, value: string) => {
    setValues((prev) => {
      const next = new Map(prev);
      next.set(id, value);
      return next;
    });
  }, []);

  return (
    <div
      style={{
        height: CONTAINER_HEIGHT,
        overflowY: 'auto',
        border: '1px solid #ccc',
      }}
      onScroll={(e) => setScrollTop(e.currentTarget.scrollTop)}
    >
      <div
        style={{
          height: items.length * ITEM_HEIGHT,
          position: 'relative',
        }}
      >
        {visible.map((item, i) => (
          <div
            key={item.id}
            style={{
              position: 'absolute',
              top: (startIndex + i) * ITEM_HEIGHT,
              height: ITEM_HEIGHT,
              display: 'flex',
              alignItems: 'center',
              padding: '0 8px',
            }}
          >
            <span style={{ marginRight: 8 }}>{item.label}</span>
            <input
              value={values.get(item.id) ?? ''}
              onChange={(e) => handleChange(item.id, e.target.value)}
            />
          </div>
        ))}
      </div>
    </div>
  );
}
```

**Asosiy nuqtalar:**

1. **`key={item.id}`** — yangi window'da match topib, item state'i to'g'ri item bilan qoladi
2. **Value parent state'da** — komponent unmount/remount bo'lsa ham, parent state saqlanadi
3. **Buffer** (`startIndex - 2`, `endIndex + 4`) — scroll smooth bo'lsin uchun ko'proq render
4. **Production** — `react-window` yoki `@tanstack/react-virtual` ishlatish tavsiya qilinadi (chunki bu sodda misol scroll restoration, dynamic height kabi murakkab case'larni qamramaydi)

To'liq kursning [`36-virtualization.md`](36-virtualization.md) bo'limi bu mavzuni chuqurroq yoritadi.

</details>

---

## Xulosa

- `Array.prototype.map` — array'ni JSX element array'iga aylantirish uchun standart vosita; `forEach` ishlamaydi (qiymat qaytarmaydi)
- `key` prop — Reconciliation algorithmiga node identity'ni xabarlash mexanizmi; bu — element'ning maxsus internal slot, prop emas
- 4 ta `key` qoidasi: **unique** (parent ichida), **stable** (render'lar bo'ylab), **predictable** (deterministic), **string/number/bigint** tip
- Index `key` sifatida **static list**'larda OK; dynamic (qo'shilish/o'chirilish/sort) yoki **stateful** item'larda anti-pattern
- `key` o'zgarishi → komponent unmount/remount; bu state reset trick uchun foydali, lekin beixtiyor o'zgarish — bug manbai
- `key` scope **parent darajasida** — boshqa parent'lardagi `key` qiymatlari bilan collision yo'q
- Reorder operationlarida stable `key` minimal DOM operation soni va to'g'ri state preservation ta'minlaydi (cross-ref [`04-reconciliation.md`](04-reconciliation.md) — `lastPlacedIndex`)
- `<>...</>` short Fragment `key` qabul qilmaydi — `<Fragment key="...">` ishlatish kerak

Keyingi bo'limda Component Basics — function components, PascalCase qoidasi, render purity invariant, va Strict Mode bilan deterministic rendering yoritiladi.

---

**Keyingi bo'lim:** [09-component-basics.md](09-component-basics.md) — Function components, PascalCase JSX transform qoidasi, render purity invariants (deterministic, no side effects, no mutable reads), idempotency va Strict Mode 2x render bilan bog'lanish.
