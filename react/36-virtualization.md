# Bo'lim 36: Virtualization

> Virtualization (yoki **windowing**) — uzun ro'yxatlar (10,000+ item) yoki katta jadvallarni render qilish uchun fundamental texnika. DOM'da faqat **ko'rinadigan element'lar** mavjud, qolganlari virtual offset orqali joylashtiriladi. Bu pattern `useRef` + `onScroll` + visible range calc bilan native React'da implement qilinishi mumkin, lekin production'da `react-window` (1D) yoki `@tanstack/react-virtual` (modern, headless, 2D) library'lari ishlatiladi. Bu fayl pure React implementation'dan tortib, library taqqoslash, variable heights, infinite scroll integration, va edge case'larni qamrab oladi.

---

## Mundarija

- [Virtualization Concept va Muammo](#virtualization-concept-va-muammo)
- [Windowing Mexanizmi](#windowing-mexanizmi)
- [Pure React Implementation — Fixed Heights](#pure-react-implementation--fixed-heights)
- [Variable Heights — Measurement Strategies](#variable-heights--measurement-strategies)
- [ResizeObserver Pattern](#resizeobserver-pattern)
- [`react-window` Library](#react-window-library)
- [`@tanstack/react-virtual` Library](#tanstackreact-virtual-library)
- [Infinite Scroll + Virtualization](#infinite-scroll--virtualization)
- [Horizontal Virtualization va 2D Grid](#horizontal-virtualization-va-2d-grid)
- [Sticky Headers va Group Items](#sticky-headers-va-group-items)
- [Use Cases](#use-cases)
- [Performance Comparison](#performance-comparison)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Virtualization Concept va Muammo

### Nazariya

Virtualization — DOM'da faqat foydalanuvchi ko'radigan element'larni saqlash strategiyasi. Yashirin (scroll'dan tashqarida) item'lar JSX'da render qilinmaydi, ularning o'rnida bo'sh joy (spacer) saqlanadi. Bu texnika browser uchun kritik resurs'larni tejaydi.

**Muammo — uzun ro'yxat naive render:**

```tsx
function ProductList({ products }: { products: Product[] }) {
  return (
    <ul>
      {products.map((product) => (
        <ProductCard key={product.id} product={product} />
      ))}
    </ul>
  );
}
```

10,000 ta `Product` bo'lsa:

1. **React Render Phase** — 10,000 ta komponent funksiyasi chaqiriladi, 10,000 ta Fiber yaratiladi → CPU intensive.
2. **Reconciliation diff** — har Fiber children diff qilinadi → O(n) Fiber tree traversal.
3. **DOM mutation** — 10,000 ta `<li>` element yaratiladi va DOM'ga qo'shiladi → DOM API call'lar.
4. **Layout/Reflow** — brauzer 10,000 element o'lchami va position'ini hisoblaydi → layout phase katta.
5. **Paint** — har element pixel'larga aylantiriladi → paint phase katta.
6. **Memory** — har DOM node Fiber + DOM struct + layout object overhead beradi (aniq miqdor browser engine'ga bog'liq).
7. **Initial scroll** — har scroll event'da layer compositing rebuild → janjitter.

**Real-world masala (sifat darajadagi taqqoslash — aniq raqamlar item complexity, browser, device'ga bog'liq):**

| | Naive (10,000 items) | Virtualization (12 visible) |
|--|---|---|
| Komponent funksiyasi chaqiriqlari | 10,000 | ~12-20 (overscan bilan) |
| DOM elementlari soni | 10,000+ | ~12-20 + container + spacer |
| React Fiber tree | ~10,000 nodes | ~12-20 nodes |
| Initial render | Sekundlar (mid-range mobile) | Frame ichida (~16ms maqsadi) |
| Memory hajmi | Yuzlab MB darajada | DOM/Fiber faqat visible items uchun |
| Scroll FPS | Past (laggy) | 60 FPS (smooth, GPU transform) |

> **Eslatma:** Yuqoridagi qiymatlar **order of magnitude** ko'rsatadi. Aniq raqamlar item komponent murakkabligi (oddiy text vs chart/image), browser engine, mobile/desktop, va React versiyasiga bog'liq. Production'da real measurement uchun `<Profiler>` API yoki DevTools Performance tab ishlatiladi (cross-ref `34-profiling.md`).

**Asosiy invariant:** Total scroll height saqlanadi (`itemCount × itemHeight`). Foydalanuvchi scroll qilganda virtual indices o'zgaradi. Ko'rinadigan range qayta render qilinadi.

> **Versiya evolyutsiyasi (Virtualization):**
> - **Pre-2015:** Custom solutions, manual scroll handler, no standard library.
> - **2015-2018 (`react-virtualized`):** Brian Vaughn library, feature-rich (List, Grid, Table, Masonry, InfiniteLoader, AutoSizer), lekin katta bundle va complex API.
> - **2018+ (`react-window`):** Brian Vaughn rewrite — lightweight ekvivalent, simpler API, faqat FixedSizeList/VariableSizeList/FixedSizeGrid/VariableSizeGrid.
> - **2022+ (`@tanstack/react-virtual`):** TanStack ekosistemasi, headless, hooks-based, 2D support, framework-agnostic core.
> - **R18+ (Concurrent):** `useTransition` smooth scroll integration, time slicing scroll handler.
> - **Sabab:** Mobile-first design, accessibility, framework-agnostic core (Vue/Svelte/Solid version'lari).

<details>
<summary><strong>Under the Hood</strong></summary>

**Browser rendering pipeline va virtualization ta'siri:**

```
JavaScript (React Render):
  ├── Without virt: 10000 Fiber, 10000 component call
  └── With virt:    12 Fiber, 12 component call

Style Recalculation:
  ├── Without virt: 10000 element CSS resolve
  └── With virt:    12 element + 1 spacer

Layout (Reflow):
  ├── Without virt: 10000 box model calculation
  └── With virt:    12 + container + spacer

Paint:
  ├── Without virt: 10000 raster
  └── With virt:    12 raster

Composite:
  ├── Without virt: GPU layer per heavy element (transform/opacity)
  └── With virt:    minimal layer count
```

**Memory layout taqqoslash (sifat darajada):**

Without virtualization: 10,000 ta React Fiber + 10,000 ta DOM node + 10,000 ta layout object — memory **item count'ga proportional**, ko'p MB darajasiga yetadi.

With virtualization: faqat visible items (overscan bilan ~12-20 ta) + container + spacer element — memory **constant** (item count o'zgaradi, lekin DOM/Fiber size deyarli o'zgarmaydi).

Memory tejash **order of magnitude** darajasida. Aniq raqamlar item komponent murakkabligi, React versiyasi, va browser engine'ga bog'liq — production'da Chrome DevTools Memory snapshot bilan tekshiriladi.

**Scroll handler invocation frequency:**

```
Native scroll fires ~60-120 events/sec on desktop
Mobile scroll fires ~30-60 events/sec

Scroll handler ish:
  1. Read scrollTop
  2. Calculate visible range (startIndex/endIndex)
  3. Compare prev range
  4. setState (if changed)
  5. React re-render visible items
  6. DOM mutation (move existing OR create new visible items)

Optimization:
  - requestAnimationFrame throttle (60fps cap)
  - Overscan (render +N items above/below visible) — scroll smooth
  - Object reuse (DOM node recycling)
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Naive (virtualization yo'q) — 10,000 item:

```tsx
function NaiveProductList({ products }: { products: Product[] }) {
  return (
    <ul style={{ height: 600, overflowY: 'auto' }}>
      {products.map((product) => (
        <li key={product.id} style={{ height: 80 }}>
          <ProductCard product={product} />
        </li>
      ))}
    </ul>
  );
}

// 10,000 products bilan:
// - Initial render: blocking
// - Memory: high
// - Scroll: laggy
```

Virtualization bilan minimal pattern:

```tsx
import { useRef, useState, useEffect } from 'react';

function VirtualizedProductList({ products }: { products: Product[] }) {
  const containerRef = useRef<HTMLUListElement>(null);
  const [scrollTop, setScrollTop] = useState(0);

  const itemHeight = 80;
  const containerHeight = 600;
  const overscan = 3;

  const visibleStart = Math.max(0, Math.floor(scrollTop / itemHeight) - overscan);
  const visibleEnd = Math.min(
    products.length,
    Math.ceil((scrollTop + containerHeight) / itemHeight) + overscan
  );

  const visibleProducts = products.slice(visibleStart, visibleEnd);
  const totalHeight = products.length * itemHeight;
  const offsetY = visibleStart * itemHeight;

  return (
    <ul
      ref={containerRef}
      onScroll={(e) => setScrollTop(e.currentTarget.scrollTop)}
      style={{ height: containerHeight, overflowY: 'auto', position: 'relative' }}
    >
      <div style={{ height: totalHeight, position: 'relative' }}>
        <div style={{ transform: `translateY(${offsetY}px)` }}>
          {visibleProducts.map((product) => (
            <li key={product.id} style={{ height: itemHeight }}>
              <ProductCard product={product} />
            </li>
          ))}
        </div>
      </div>
    </ul>
  );
}

// 10,000 products bilan:
// - Initial render: ~50ms
// - Memory: low (~12 DOM nodes)
// - Scroll: smooth
```

</details>

---

## Windowing Mexanizmi

### Nazariya

Windowing — virtualization'ning yadro algoritmi. To'rtta asosiy parametr orqali **visible range**'ni hisoblaydi:

1. **`scrollTop`** — container'ning vertical scroll position'i (pikselda).
2. **`itemHeight`** — har bir item balandligi (fixed yoki measured).
3. **`containerHeight`** — viewport balandligi.
4. **`itemCount`** — umumiy item soni.

**Visible range formula** (fixed heights):

```
startIndex = Math.floor(scrollTop / itemHeight)
endIndex   = Math.ceil((scrollTop + containerHeight) / itemHeight)
visibleCount = endIndex - startIndex
```

**Overscan** — visible range tashqarisida qo'shimcha item'larni render qilish (default 1-5). Sabab:

- Foydalanuvchi tez scroll qilsa — visible range hali render bo'lmagan element'larga yetishi mumkin → "blank" effect
- Scroll handler async — paint paytida brauzer eski DOM ko'rsatadi
- Overscan bilan smooth perception, lekin memory + render cost biroz oshadi

**Spacer pattern** (total height saqlash):

Virtualization scroll bar position'i va proporsiyasini saqlash uchun container ichida total height bilan spacer element zarur:

```
Container (overflowY: auto, height: 600)
  └── Spacer (height: itemCount × itemHeight)
        └── Render Layer (transform: translateY(offsetY))
              └── Visible items
```

**Offset calculation:**

```
offsetY = startIndex × itemHeight
transform = translateY(offsetY) → ko'rinadigan items o'zlarining haqiqiy position'iga
```

**Key reuse:** Virtualization'da `key={index}` ishlatish **NOTO'G'RI** — har scroll'da indices o'zgaradi va React Reconciler item'larni unmount/mount qiladi (state lost, Fiber rebuild). Item'ning unique ID ishlatish kerak (cross-ref `08-list-rendering.md`).

<details>
<summary><strong>Under the Hood</strong></summary>

**Visible range calculation timeline (60 FPS scroll):**

```
Frame 0 (t=0ms):    scrollTop = 0       → range [0, 9]
Frame 1 (t=16ms):   scrollTop = 80      → range [1, 10]
Frame 2 (t=32ms):   scrollTop = 160     → range [2, 11]
Frame 3 (t=48ms):   scrollTop = 240     → range [3, 12]
...

Per frame:
  1. onScroll fires (passive event)
  2. setState(scrollTop) — schedule re-render
  3. React commits → calculate range
  4. React reconcile visible items
  5. DOM mutation: existing items reuse (key=id), new items create
  6. Browser paints
```

**`transform: translateY` GPU-acceleration:**

```
CSS transform → composite layer
  → GPU translateY(80px) per frame
  → Paint avoided (no repaint of items)
  → Smooth 60 FPS

Alternative: top: 80px → layout invalidation per frame → poor perf
```

**Overscan trade-off:**

```
overscan = 0:
  Memory minimal: ~visibleCount items
  Scroll lag risk: rendering during scroll

overscan = 3:
  Memory: visibleCount + 6 items
  Scroll smooth: pre-rendered above/below

overscan = 10:
  Memory: visibleCount + 20 items
  Scroll very smooth, but memory cost

Default: 1-5 (library defaults)
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Visible range calculation isolated function:

```tsx
function calculateVisibleRange(
  scrollTop: number,
  itemHeight: number,
  containerHeight: number,
  itemCount: number,
  overscan = 3
): { start: number; end: number; offsetY: number; totalHeight: number } {
  const visibleStart = Math.floor(scrollTop / itemHeight);
  const visibleEnd = Math.ceil((scrollTop + containerHeight) / itemHeight);

  const start = Math.max(0, visibleStart - overscan);
  const end = Math.min(itemCount, visibleEnd + overscan);

  return {
    start,
    end,
    offsetY: start * itemHeight,
    totalHeight: itemCount * itemHeight,
  };
}

// Misol:
const range = calculateVisibleRange(800, 80, 600, 10000, 3);
// { start: 7, end: 21, offsetY: 560, totalHeight: 800000 }
```

requestAnimationFrame throttle scroll handler:

```tsx
import { useEffect, useRef, useState } from 'react';

function useThrottledScroll(initialScrollTop = 0) {
  const [scrollTop, setScrollTop] = useState(initialScrollTop);
  const rafIdRef = useRef<number | null>(null);
  const latestScrollTopRef = useRef(initialScrollTop);

  const handleScroll = (e: React.UIEvent<HTMLElement>) => {
    latestScrollTopRef.current = e.currentTarget.scrollTop;

    if (rafIdRef.current !== null) return;

    rafIdRef.current = requestAnimationFrame(() => {
      setScrollTop(latestScrollTopRef.current);
      rafIdRef.current = null;
    });
  };

  useEffect(() => {
    return () => {
      if (rafIdRef.current !== null) {
        cancelAnimationFrame(rafIdRef.current);
      }
    };
  }, []);

  return [scrollTop, handleScroll] as const;
}
```

Spacer pattern visualization:

```tsx
function SpacerPatternExample() {
  const [scrollTop, setScrollTop] = useState(0);
  const itemCount = 10000;
  const itemHeight = 80;
  const containerHeight = 600;

  const { start, end, offsetY, totalHeight } = calculateVisibleRange(
    scrollTop, itemHeight, containerHeight, itemCount, 3
  );

  return (
    <div
      onScroll={(e) => setScrollTop(e.currentTarget.scrollTop)}
      style={{
        height: containerHeight,
        overflowY: 'auto',
        border: '1px solid #ccc',
      }}
    >
      <div style={{ height: totalHeight, position: 'relative' }}>
        <div
          style={{
            position: 'absolute',
            top: 0,
            left: 0,
            right: 0,
            transform: `translateY(${offsetY}px)`,
          }}
        >
          {Array.from({ length: end - start }, (_, i) => {
            const index = start + i;
            return (
              <div key={index} style={{ height: itemHeight }}>
                Item {index}
              </div>
            );
          })}
        </div>
      </div>
    </div>
  );
}
```

</details>

---

## Pure React Implementation — Fixed Heights

### Nazariya

Pure React implementation — library'siz, faqat React primitives bilan virtualization. Fixed heights (har item bir xil balandlikda) holatda algorithm soddaroq. Pattern muhim — library tushunish va custom case'lar uchun.

**Komponent dizayni:**

- **Generic typing** — `T` item type bo'yicha generic.
- **Render prop** — har item uchun render funksiyasi (`renderItem(item, index) => ReactNode`).
- **Stable scroll handler** — `useCallback` + `requestAnimationFrame` throttle.
- **Container ref** — scroll API'lariga access.
- **`scrollToIndex`** programmatic API (imperative handle).

**API design pattern:**

```tsx
<VirtualList
  items={products}
  itemHeight={80}
  height={600}
  overscan={3}
  renderItem={(product, index) => <ProductCard key={product.id} product={product} />}
/>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Strict Mode 2x render impact:**

R18+ Strict Mode dev'da `useState` + `useEffect` 2x cycle. Virtualization'da:

- `setScrollTop` 2x → calculate range 2x → render 2x
- `useEffect` (resize listener) — mount → cleanup → mount

Bu development-only behavior. Lekin 10,000+ items + 2x render measurement haqiqiy emas. Production build profiling kerak (cross-ref `34-profiling.md`).

**Concurrent rendering bilan integration:**

R18+ `useTransition` scroll'ni non-urgent qilishi mumkin:

```tsx
const [isPending, startTransition] = useTransition();

const handleScroll = (e: React.UIEvent<HTMLElement>) => {
  startTransition(() => {
    setScrollTop(e.currentTarget.scrollTop);
  });
};
```

Lekin scroll user interaction → urgent qilishi tabiiy. `startTransition` faqat heavy item'lar render bo'lganda foydali (rich content + chart per item). Default urgent saqlash tavsiya.

**`scrollToIndex` mechanism:**

```
scrollToIndex(index, align = 'start')
  → newScrollTop = align === 'start'   ? index × itemHeight
                 : align === 'center'  ? index × itemHeight - (containerHeight - itemHeight) / 2
                 : align === 'end'     ? (index + 1) × itemHeight - containerHeight
                 : align === 'auto'    ? // nearest visible adjustment
  → containerRef.current.scrollTop = newScrollTop
  → Browser onScroll fires → setState → re-render
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

To'liq pure React virtualization komponenti:

```tsx
import {
  useCallback,
  useEffect,
  useImperativeHandle,
  useRef,
  useState,
  forwardRef,
  type ReactNode,
  type UIEvent,
} from 'react';

interface VirtualListHandle {
  scrollToIndex: (index: number, align?: 'start' | 'center' | 'end') => void;
  scrollToTop: () => void;
}

interface VirtualListProps<T> {
  items: T[];
  itemHeight: number;
  height: number;
  overscan?: number;
  renderItem: (item: T, index: number) => ReactNode;
  onScroll?: (scrollTop: number) => void;
  className?: string;
}

function VirtualListInner<T>(
  {
    items,
    itemHeight,
    height,
    overscan = 3,
    renderItem,
    onScroll,
    className,
  }: VirtualListProps<T>,
  ref: React.Ref<VirtualListHandle>
) {
  const containerRef = useRef<HTMLDivElement>(null);
  const [scrollTop, setScrollTop] = useState(0);
  const rafIdRef = useRef<number | null>(null);
  const latestScrollTopRef = useRef(0);

  const itemCount = items.length;
  const totalHeight = itemCount * itemHeight;
  const visibleCount = Math.ceil(height / itemHeight);

  const startIndex = Math.max(0, Math.floor(scrollTop / itemHeight) - overscan);
  const endIndex = Math.min(itemCount, startIndex + visibleCount + overscan * 2);
  const offsetY = startIndex * itemHeight;

  const handleScroll = useCallback(
    (e: UIEvent<HTMLDivElement>) => {
      latestScrollTopRef.current = e.currentTarget.scrollTop;

      if (rafIdRef.current !== null) return;

      rafIdRef.current = requestAnimationFrame(() => {
        setScrollTop(latestScrollTopRef.current);
        onScroll?.(latestScrollTopRef.current);
        rafIdRef.current = null;
      });
    },
    [onScroll]
  );

  useEffect(() => {
    return () => {
      if (rafIdRef.current !== null) {
        cancelAnimationFrame(rafIdRef.current);
      }
    };
  }, []);

  useImperativeHandle(
    ref,
    () => ({
      scrollToIndex: (index, align = 'start') => {
        if (!containerRef.current) return;

        let target: number;
        switch (align) {
          case 'start':
            target = index * itemHeight;
            break;
          case 'center':
            target = index * itemHeight - (height - itemHeight) / 2;
            break;
          case 'end':
            target = (index + 1) * itemHeight - height;
            break;
        }

        containerRef.current.scrollTop = Math.max(0, Math.min(totalHeight - height, target));
      },
      scrollToTop: () => {
        if (containerRef.current) {
          containerRef.current.scrollTop = 0;
        }
      },
    }),
    [itemHeight, height, totalHeight]
  );

  const visibleItems: ReactNode[] = [];
  for (let i = startIndex; i < endIndex; i++) {
    visibleItems.push(renderItem(items[i], i));
  }

  return (
    <div
      ref={containerRef}
      onScroll={handleScroll}
      className={className}
      style={{
        height,
        overflowY: 'auto',
        position: 'relative',
        WebkitOverflowScrolling: 'touch',
      }}
    >
      <div style={{ height: totalHeight, position: 'relative' }}>
        <div
          style={{
            position: 'absolute',
            top: 0,
            left: 0,
            right: 0,
            transform: `translateY(${offsetY}px)`,
          }}
        >
          {visibleItems}
        </div>
      </div>
    </div>
  );
}

export const VirtualList = forwardRef(VirtualListInner) as <T>(
  props: VirtualListProps<T> & { ref?: React.Ref<VirtualListHandle> }
) => ReactNode;
```

Ishlatish — ProductCard list:

```tsx
function ProductsPage({ products }: { products: Product[] }) {
  const listRef = useRef<VirtualListHandle>(null);

  return (
    <div>
      <button onClick={() => listRef.current?.scrollToTop()}>
        Tepaga qaytish
      </button>

      <VirtualList<Product>
        ref={listRef}
        items={products}
        itemHeight={80}
        height={600}
        overscan={5}
        renderItem={(product) => (
          <div
            key={product.id}
            style={{
              height: 80,
              borderBottom: '1px solid #eee',
              padding: 12,
            }}
          >
            <h3>{product.name}</h3>
            <p>${product.price}</p>
          </div>
        )}
      />
    </div>
  );
}
```

`useTransition` bilan scroll smoothing (heavy item rendering):

```tsx
import { useTransition } from 'react';

function HeavyItemList({ items }: { items: HeavyItem[] }) {
  const [isPending, startTransition] = useTransition();
  const [scrollTop, setScrollTop] = useState(0);

  const handleScroll = (e: React.UIEvent<HTMLDivElement>) => {
    startTransition(() => {
      setScrollTop(e.currentTarget.scrollTop);
    });
  };

  return (
    <div
      onScroll={handleScroll}
      style={{ height: 600, overflow: 'auto', opacity: isPending ? 0.7 : 1 }}
    >
      {/* Virtualization render */}
    </div>
  );
}
```

</details>

---

## Variable Heights — Measurement Strategies

### Nazariya

Variable heights — har item turli balandlikda bo'lishi mumkin (chat message'lar, dynamic content, expandable cards). Bu fixed heights formula'sini buzadi: `index × itemHeight` aniq emas.

**Strategy variantlari:**

1. **Pre-known heights array** — har item uchun explicit balandlik mavjud (oldindan hisoblangan, server'dan kelgan). Eng samarali.
2. **Estimate then measure** — initial render paytida har item taxminiy balandlikda joylashtiriladi, layout'dan keyin `ResizeObserver` haqiqiy heightlarni o'lchaydi va offset cache yangilaydi.
3. **Single fixed height with override** — default fixed, ba'zi item'lar custom height'ga ega.

**Estimate then measure pattern:**

- Item render bo'lguncha estimate (50px, 100px misol)
- Render bo'lgach `ResizeObserver` haqiqiy height aniqlaydi
- Offset cache obyekt yangilanadi (`Map<index, { offset, height }>`)
- Subsequent scroll'da cache'dan foydalanadi
- Measured items uchun aniq offset, hali measure bo'lmagan items uchun fallback estimate

**Cumulative offset calculation:**

```
items: [10, 20, 30, 40, 50]
heights: [50, 80, 120, 60, 90]
offsets: [0, 50, 130, 250, 310]  ← prefix sum
totalHeight: 400
```

`offsets[i] = offsets[i-1] + heights[i-1]`. Variable heights virtualization library'lari bu hisob'ni cache'da saqlaydi.

**Binary search find startIndex:**

`scrollTop` qaysi item index'ga to'g'ri kelishini topish:

```
Linear: O(n) — sequential offsets check
Binary search: O(log n) — sorted offsets

10,000 items'da:
  Linear: 10,000 comparison
  Binary: 13 comparison
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Offset cache data structure:**

```typescript
class OffsetCache {
  private heights: number[] = [];
  private offsets: number[] = [];
  private totalHeight = 0;

  setHeight(index: number, height: number): void {
    if (this.heights[index] === height) return;

    this.heights[index] = height;
    this.recomputeOffsetsFrom(index);
  }

  private recomputeOffsetsFrom(startIndex: number): void {
    let offset = startIndex === 0 ? 0 : this.offsets[startIndex - 1] + this.heights[startIndex - 1];

    for (let i = startIndex; i < this.heights.length; i++) {
      this.offsets[i] = offset;
      offset += this.heights[i];
    }

    this.totalHeight = offset;
  }

  findIndexForOffset(scrollTop: number): number {
    let lo = 0;
    let hi = this.offsets.length - 1;

    while (lo <= hi) {
      const mid = (lo + hi) >>> 1;
      const midOffset = this.offsets[mid];

      if (midOffset === scrollTop) return mid;
      if (midOffset < scrollTop) lo = mid + 1;
      else hi = mid - 1;
    }

    return Math.max(0, lo - 1);
  }

  getOffset(index: number): number {
    return this.offsets[index] ?? 0;
  }

  getHeight(index: number): number {
    return this.heights[index] ?? 0;
  }

  getTotalHeight(): number {
    return this.totalHeight;
  }
}
```

**Estimate vs measured trade-off:**

```
All items measured:
  Total height: accurate
  Scroll bar: accurate proportions

Some items not yet measured (estimate fallback):
  Total height: estimate-based
  Scroll bar: jumpy when items measure
  User experience: scroll bar position adjusts as scroll progresses

Solution: estimateSize callback closer to real average
  - Chat messages: avg ~50px
  - Comments: avg ~120px
  - Product cards: avg ~80px
```

**ResizeObserver entries va batching:**

```javascript
const observer = new ResizeObserver((entries) => {
  // entries — barcha o'lchangan element'lar
  for (const entry of entries) {
    const index = parseInt(entry.target.dataset.index!, 10);
    const newHeight = entry.contentRect.height;
    cache.setHeight(index, newHeight);
  }

  // Bitta setState barcha update'lar uchun (batched)
  triggerRerender();
});

// Har item observe:
observer.observe(itemElement);
```

ResizeObserver browser API — async va batched. Spec'ga ko'ra callback **har frame'ning layout fazasidan keyin, paint'dan oldin** chaqiriladi (microtask'lar yoki idle queue emas — layout pipeline'ning maxsus bosqichida). Bitta frame'da bir nechta element o'zgarsa, callback bitta `entries` array bilan yagona marta chaqiriladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Pre-known heights bilan variable list:

```tsx
interface VariableListProps<T> {
  items: T[];
  itemHeights: number[]; // Har item uchun balandlik
  height: number;
  overscan?: number;
  renderItem: (item: T, index: number) => ReactNode;
}

function VariableList<T>({
  items,
  itemHeights,
  height,
  overscan = 3,
  renderItem,
}: VariableListProps<T>) {
  const [scrollTop, setScrollTop] = useState(0);

  const offsets = useMemo(() => {
    const result = new Array(items.length);
    let acc = 0;
    for (let i = 0; i < items.length; i++) {
      result[i] = acc;
      acc += itemHeights[i];
    }
    return { offsets: result, totalHeight: acc };
  }, [items.length, itemHeights]);

  const findStartIndex = (top: number): number => {
    let lo = 0;
    let hi = offsets.offsets.length - 1;
    while (lo <= hi) {
      const mid = (lo + hi) >>> 1;
      if (offsets.offsets[mid] < top) lo = mid + 1;
      else hi = mid - 1;
    }
    return Math.max(0, lo - 1);
  };

  const startIndex = Math.max(0, findStartIndex(scrollTop) - overscan);

  let endIndex = startIndex;
  let acc = offsets.offsets[startIndex];
  while (endIndex < items.length && acc < scrollTop + height) {
    acc += itemHeights[endIndex];
    endIndex++;
  }
  endIndex = Math.min(items.length, endIndex + overscan);

  const offsetY = offsets.offsets[startIndex];

  return (
    <div
      onScroll={(e) => setScrollTop(e.currentTarget.scrollTop)}
      style={{ height, overflowY: 'auto', position: 'relative' }}
    >
      <div style={{ height: offsets.totalHeight, position: 'relative' }}>
        <div
          style={{
            position: 'absolute',
            top: 0,
            left: 0,
            right: 0,
            transform: `translateY(${offsetY}px)`,
          }}
        >
          {Array.from({ length: endIndex - startIndex }, (_, i) => {
            const index = startIndex + i;
            return (
              <div key={index} style={{ height: itemHeights[index] }}>
                {renderItem(items[index], index)}
              </div>
            );
          })}
        </div>
      </div>
    </div>
  );
}
```

Ishlatish — Comment list bilan variable heights:

```tsx
function CommentsList({ comments }: { comments: Comment[] }) {
  const heights = useMemo(
    () => comments.map((c) => 60 + Math.ceil(c.text.length / 50) * 20),
    [comments]
  );

  return (
    <VariableList
      items={comments}
      itemHeights={heights}
      height={500}
      renderItem={(comment) => (
        <div style={{ padding: 12, borderBottom: '1px solid #eee' }}>
          <strong>{comment.author}</strong>
          <p>{comment.text}</p>
        </div>
      )}
    />
  );
}
```

</details>

---

## ResizeObserver Pattern

### Nazariya

`ResizeObserver` — DOM API element'ning o'lcham o'zgarishlarini kuzatadi. Variable heights virtualization'da kritik: render bo'lgan har item haqiqiy balandligini measure qilib offset cache'ga yozadi.

**Browser support:**

- Chrome 64+, Firefox 69+, Safari 13.1+, Edge 79+
- Mobile: iOS Safari 13.4+, Chrome Android 64+
- IE: support yo'q (polyfill kerak edi)

**Lifecycle:**

```
Item mount → useEffect → observer.observe(element)
Item unmount → useEffect cleanup → observer.unobserve(element)
Element resize → callback fires → cache update → trigger re-render
```

**Batching:**

`ResizeObserver` callback brauzer'da batched. Bir nechta element bir vaqtda o'zgarsa, callback bitta entries array bilan chaqiriladi. React state update ham batched (R18+).

**Single shared observer pattern:**

Har item uchun alohida `ResizeObserver` overhead. Yaxshiroq pattern: bitta observer barcha items uchun, `dataset.index` orqali map qilinadi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`ResizeObserver` callback specification:**

```javascript
new ResizeObserver((entries, observer) => {
  // entries: ResizeObserverEntry[]
  // entry.target: Element
  // entry.contentRect: DOMRectReadOnly { width, height, x, y, ... }
  // entry.borderBoxSize: ResizeObserverSize[] (newer API)
  // entry.contentBoxSize: ResizeObserverSize[]
  // entry.devicePixelContentBoxSize: ResizeObserverSize[] (Safari 16+)
});
```

**Observer notification timing (spec: "deliver resize loop notifications" step):**

```
1. Layout phase complete (style + layout computed)
2. Browser collects elements whose size changed
3. "deliver resize loop notifications" step runs — alohida queue,
   microtask queue YOK'O idle queue EMAS (spec'da maxsus step)
4. Callback fires synchronously in this step, before paint
5. setState in callback → React schedules re-render (next frame)
6. Next frame: new layout cycle
```

**MUHIM:** Bu callback "microtask after layout" emas — bu rendering pipeline'da layout va paint orasida joylashgan **alohida bosqich** (CSS Resize Observer spec § 3.5 "Procedure"). Shu sababli ResizeObserver callback'da DOM read'lar (`getBoundingClientRect`) layout reflow keltirib chiqarmaydi — layout endigina hisoblangan.

**Loop prevention:**

`ResizeObserver` callback ichida element o'lchami o'zgartirilsa (`element.style.height = ...`), brauzer infinite loop'ni oldini olish uchun keyingi frame'gacha kechiktiradi (skip va console error: "ResizeObserver loop limit exceeded").

**Memory considerations (sifat darajada):**

- Har observed element observer ichida reference saqlaydi — kichik overhead per element (browser implementation'ga bog'liq).
- 1000+ element kuzatilganda umumiy overhead sezilarli, lekin item'larning o'zlari DOM/Fiber yo'qligi tufayli virtualization umumiy memory'ni sezilarli darajada kamaytiradi.

**Single shared observer afzalligi:**

- 1 ta `ResizeObserver` instance bilan barcha items kuzatilsa: 1 ta callback batched entries bilan, optimal.
- Har item alohida observer: N ta callback chaqiriqlari, ko'p memory + CPU overhead.

Best practice — module-level yoki ref-stable yagona shared observer.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Single shared `ResizeObserver` hook:

```tsx
import { useEffect, useRef, useCallback } from 'react';

export function useSharedResizeObserver(
  onResize: (index: number, height: number) => void
) {
  const observerRef = useRef<ResizeObserver | null>(null);
  const elementsRef = useRef<Map<Element, number>>(new Map());

  if (observerRef.current === null && typeof window !== 'undefined') {
    observerRef.current = new ResizeObserver((entries) => {
      for (const entry of entries) {
        const index = elementsRef.current.get(entry.target);
        if (index !== undefined) {
          onResize(index, entry.contentRect.height);
        }
      }
    });
  }

  const observe = useCallback((element: Element | null, index: number) => {
    if (!element || !observerRef.current) return;

    elementsRef.current.set(element, index);
    observerRef.current.observe(element);

    return () => {
      observerRef.current?.unobserve(element);
      elementsRef.current.delete(element);
    };
  }, []);

  useEffect(() => {
    return () => {
      observerRef.current?.disconnect();
      observerRef.current = null;
      elementsRef.current.clear();
    };
  }, []);

  return observe;
}
```

To'liq variable heights virtualization with measurement:

```tsx
import { useCallback, useEffect, useRef, useState, useMemo } from 'react';

interface DynamicVirtualListProps<T> {
  items: T[];
  estimateSize: (index: number) => number;
  height: number;
  overscan?: number;
  renderItem: (item: T, index: number) => ReactNode;
  getItemKey: (item: T, index: number) => string | number;
}

function DynamicVirtualList<T>({
  items,
  estimateSize,
  height,
  overscan = 3,
  renderItem,
  getItemKey,
}: DynamicVirtualListProps<T>) {
  const [scrollTop, setScrollTop] = useState(0);
  const [, forceUpdate] = useState(0);

  const heightsRef = useRef<Map<number, number>>(new Map());
  const containerRef = useRef<HTMLDivElement>(null);

  const totalHeight = useMemo(() => {
    let total = 0;
    for (let i = 0; i < items.length; i++) {
      total += heightsRef.current.get(i) ?? estimateSize(i);
    }
    return total;
  }, [items.length, estimateSize]);

  const findRange = useCallback(() => {
    let acc = 0;
    let startIndex = 0;
    let endIndex = 0;

    for (let i = 0; i < items.length; i++) {
      const itemHeight = heightsRef.current.get(i) ?? estimateSize(i);

      if (acc + itemHeight > scrollTop && startIndex === 0 && acc <= scrollTop) {
        startIndex = i;
      }

      if (acc < scrollTop + height) {
        endIndex = i + 1;
      }

      acc += itemHeight;
    }

    return {
      start: Math.max(0, startIndex - overscan),
      end: Math.min(items.length, endIndex + overscan),
    };
  }, [items.length, scrollTop, height, overscan, estimateSize]);

  const { start, end } = findRange();

  const offsetY = useMemo(() => {
    let acc = 0;
    for (let i = 0; i < start; i++) {
      acc += heightsRef.current.get(i) ?? estimateSize(i);
    }
    return acc;
  }, [start, estimateSize]);

  const handleResize = useCallback((index: number, newHeight: number) => {
    const prevHeight = heightsRef.current.get(index);
    if (prevHeight !== newHeight) {
      heightsRef.current.set(index, newHeight);
      forceUpdate((n) => n + 1);
    }
  }, []);

  const observe = useSharedResizeObserver(handleResize);

  return (
    <div
      ref={containerRef}
      onScroll={(e) => setScrollTop(e.currentTarget.scrollTop)}
      style={{ height, overflowY: 'auto', position: 'relative' }}
    >
      <div style={{ height: totalHeight, position: 'relative' }}>
        <div
          style={{
            position: 'absolute',
            top: 0,
            left: 0,
            right: 0,
            transform: `translateY(${offsetY}px)`,
          }}
        >
          {Array.from({ length: end - start }, (_, i) => {
            const index = start + i;
            const item = items[index];

            return (
              <div
                key={getItemKey(item, index)}
                ref={(el) => observe(el, index)}
                data-index={index}
              >
                {renderItem(item, index)}
              </div>
            );
          })}
        </div>
      </div>
    </div>
  );
}
```

Ishlatish — chat messages variable height:

```tsx
function ChatMessages({ messages }: { messages: ChatMessage[] }) {
  return (
    <DynamicVirtualList
      items={messages}
      estimateSize={() => 60}
      height={500}
      overscan={5}
      getItemKey={(msg) => msg.id}
      renderItem={(msg) => (
        <div style={{ padding: 12, borderBottom: '1px solid #eee' }}>
          <strong>{msg.author}</strong>
          <p style={{ whiteSpace: 'pre-wrap' }}>{msg.text}</p>
          {msg.attachment && <img src={msg.attachment} alt="" />}
        </div>
      )}
    />
  );
}
```

</details>

---

## `react-window` Library

### Nazariya

`react-window` — Bvaughn (Brian Vaughn, ex-React core team) tomonidan yozilgan minimalistic virtualization library. `react-virtualized` o'rniga zamonaviy alternative.

**Asosiy export'lar:**

- **`FixedSizeList`** — fixed item height, vertical yoki horizontal.
- **`VariableSizeList`** — variable heights, har index uchun explicit `itemSize(index)` callback.
- **`FixedSizeGrid`** — fixed cell size 2D grid.
- **`VariableSizeGrid`** — variable cell sizes 2D grid.
- **`areEqual`** — `React.memo` shallow comparison helper item komponent'i uchun.

**Bundle size:** Lightweight (kichik gzipped hajm — aniq raqam uchun `bundlephobia.com/package/react-window`).

**Limitations:**

- Variable heights uchun explicit `itemSize` kerak. Pre-known emas variable case'lar uchun manual measurement kerak.
- 2D virtualization faqat grid (rows + cols), dynamic table layout cheklangan.
- Fizikaviy DOM scroll, `scrollIntoView` API mavjud emas.

**Anatomy of `FixedSizeList`:**

```tsx
<FixedSizeList
  height={500}        // Container height
  width={300}         // Container width
  itemCount={10000}   // Total items
  itemSize={50}       // Each item height (or width if horizontal)
  overscanCount={5}   // Render extra items above/below visible
  layout="vertical"   // 'vertical' | 'horizontal'
>
  {Row} {/* Children render function */}
</FixedSizeList>
```

`Row` komponenti `({ index, style })` props oladi va **`style`'ni o'z element'ga uzatishi MAJBURIY** (library transform translateY orqali position'ni belgilaydi).

> **⚠️ MUHIM — Row komponentni module-level define qilish:** `<FixedSizeList>{({index, style}) => <div>...</div>}</FixedSizeList>` shaklida **inline arrow function** ishlatilsa, har parent render'da yangi component reference yaratiladi. Bu Reconciler uchun **yangi komponent turi** kabi ko'rinadi → barcha visible item'lar har render'da unmount/mount qilinadi (state lost, ResizeObserver attach/detach, DOM rebuild). To'g'ri pattern: alohida `const Row = memo(({...}: RowProps) => ...)` define qilib `{Row}` pass qilish. Quyidagi misollardagi inline'lar **soddalashtirilgan** — production'da har doim `Row` alohida.

<details>
<summary><strong>Kod Misollari</strong></summary>

`FixedSizeList` minimal misol:

```tsx
import { FixedSizeList, type ListChildComponentProps } from 'react-window';

interface RowProps extends ListChildComponentProps {
  data: Product[];
}

function Row({ index, style, data }: RowProps) {
  const product = data[index];

  return (
    <div style={style}>
      <h3>{product.name}</h3>
      <p>${product.price}</p>
    </div>
  );
}

function ProductsListVirtualized({ products }: { products: Product[] }) {
  return (
    <FixedSizeList
      height={600}
      width="100%"
      itemCount={products.length}
      itemSize={80}
      itemData={products}
      overscanCount={5}
    >
      {Row}
    </FixedSizeList>
  );
}
```

`React.memo` + `areEqual` performance pattern:

```tsx
import { FixedSizeList, areEqual } from 'react-window';
import { memo } from 'react';

const Row = memo(({ index, style, data }: RowProps) => {
  const product = data[index];

  return (
    <div style={style}>
      <h3>{product.name}</h3>
      <p>${product.price}</p>
    </div>
  );
}, areEqual);

// areEqual — shallow comparison + style reference check
// React.memo default shallow comparison ham ishlaydi, lekin areEqual library-aware
```

`VariableSizeList` — explicit heights:

```tsx
import { VariableSizeList } from 'react-window';
import { useRef } from 'react';

function CommentsListVirtualized({ comments }: { comments: Comment[] }) {
  const listRef = useRef<VariableSizeList>(null);

  const getItemSize = (index: number): number => {
    const comment = comments[index];
    const baseHeight = 60;
    const lineHeight = 20;
    const charsPerLine = 50;
    const lines = Math.ceil(comment.text.length / charsPerLine);
    return baseHeight + lines * lineHeight;
  };

  return (
    <VariableSizeList
      ref={listRef}
      height={500}
      width="100%"
      itemCount={comments.length}
      itemSize={getItemSize}
      itemData={comments}
      overscanCount={3}
    >
      {({ index, style, data }) => (
        <div style={style}>
          <strong>{data[index].author}</strong>
          <p>{data[index].text}</p>
        </div>
      )}
    </VariableSizeList>
  );
}
```

`scrollToItem` programmatic API:

```tsx
import { FixedSizeList } from 'react-window';
import { useRef } from 'react';

function ChatListWithScrollControl({ messages }: { messages: Message[] }) {
  const listRef = useRef<FixedSizeList>(null);

  const scrollToBottom = () => {
    listRef.current?.scrollToItem(messages.length - 1, 'end');
  };

  const scrollToMessage = (id: string) => {
    const index = messages.findIndex((m) => m.id === id);
    if (index !== -1) {
      listRef.current?.scrollToItem(index, 'center');
    }
  };

  return (
    <>
      <button onClick={scrollToBottom}>Pastga</button>

      <FixedSizeList
        ref={listRef}
        height={500}
        width="100%"
        itemCount={messages.length}
        itemSize={60}
      >
        {({ index, style }) => (
          <div style={style}>{messages[index].text}</div>
        )}
      </FixedSizeList>
    </>
  );
}
```

`FixedSizeGrid` — 2D grid (e.g. photo gallery):

```tsx
import { FixedSizeGrid, type GridChildComponentProps } from 'react-window';

interface PhotoCellProps extends GridChildComponentProps {
  data: Photo[][];
}

function PhotoCell({ columnIndex, rowIndex, style, data }: PhotoCellProps) {
  const photo = data[rowIndex][columnIndex];
  if (!photo) return <div style={style} />;

  return (
    <div style={style}>
      <img src={photo.thumbnail} alt={photo.title} />
    </div>
  );
}

function PhotoGallery({ photos }: { photos: Photo[] }) {
  const columnCount = 4;
  const rows: Photo[][] = [];

  for (let i = 0; i < photos.length; i += columnCount) {
    rows.push(photos.slice(i, i + columnCount));
  }

  return (
    <FixedSizeGrid
      columnCount={columnCount}
      columnWidth={200}
      rowCount={rows.length}
      rowHeight={200}
      height={600}
      width={800}
      itemData={rows}
    >
      {PhotoCell}
    </FixedSizeGrid>
  );
}
```

Horizontal layout:

```tsx
<FixedSizeList
  layout="horizontal"
  height={150}
  width={800}
  itemCount={50}
  itemSize={120}
>
  {({ index, style }) => (
    <div style={style}>
      <ImageCard imageId={index} />
    </div>
  )}
</FixedSizeList>
```

</details>

---

## `@tanstack/react-virtual` Library

### Nazariya

`@tanstack/react-virtual` — TanStack ekosistemasi (React Query, React Table, React Router author Tanner Linsley). Modern alternative `react-window`'ga, lekin **headless** va **hooks-based**.

**Asosiy farqlar `react-window`'dan:**

| Aspect | react-window | @tanstack/react-virtual |
|--------|--------------|-------------------------|
| API style | Component-based | Hooks-based (headless) |
| Variable heights | Explicit `itemSize(index)` | Automatic measurement (ResizeObserver) |
| 2D support | Grid only | Full 2D (rows + cols mustaqil) |
| Bundle size | Lightweight (gzipped) | Lightweight (gzipped, biroz kattaroq) |
| TypeScript | OK | First-class |
| Framework support | React only | React, Vue, Svelte, Solid (core agnostic) |
| Dynamic content | Manual `resetAfterIndex` | Automatic via ResizeObserver |
| Scroll API | `scrollToItem` | `scrollToIndex`, `scrollToOffset` |

**`useVirtualizer` hook** asosiy API:

```tsx
const virtualizer = useVirtualizer({
  count: items.length,
  getScrollElement: () => parentRef.current,
  estimateSize: () => 50,
  overscan: 5,
});
```

**Returns:**

- `virtualizer.getVirtualItems()` — visible items array (`{ index, start, size, end, key, lane }`)
- `virtualizer.getTotalSize()` — total scroll height (px)
- `virtualizer.scrollToIndex(index, options?): void` — programmatic scroll (sync, return value yo'q)
- `virtualizer.measureElement` — **ref callback** (har item'ga `ref={virtualizer.measureElement}` ulanadi, library ResizeObserver attach qiladi)
- `virtualizer.resizeItem(item: VirtualItem, size: number): void` — manual size override (VirtualItem object qabul qiladi, index emas)

**Key benefit — automatic measurement:**

`measureElement` ref callback har item'ga ulansa, library `ResizeObserver` orqali haqiqiy heightlarni o'lchaydi va offset cache yangilaydi. Variable heights uchun manual measurement kerak emas.

<details>
<summary><strong>Under the Hood</strong></summary>

**`useVirtualizer` internal state machine:**

```
Initial:
  - count = 1000
  - estimateSize = 50 (callback)
  - measurements: Map<index, { start, size, end }>

First render:
  - All indices use estimateSize → 50px each
  - Total height: 50,000

After mount:
  - getVirtualItems() returns visible range
  - Each item rendered with ref={virtualizer.measureElement}
  - ResizeObserver attached
  - Real heights measured asynchronously

After measurement:
  - measurements Map updated
  - Total height recalculated
  - getVirtualItems() returns adjusted positions
  - Subsequent scrolls use cached heights
```

**`getVirtualItems` return shape:**

```typescript
interface VirtualItem {
  index: number;       // Original item index
  start: number;       // Top offset in pixels
  size: number;        // Item height
  end: number;         // start + size
  key: number | string; // Stable key
  lane: number;        // Column index (for masonry/grid)
}
```

**Comparison: react-window auto-measurement workaround:**

```tsx
// react-window — manual workaround for auto-measure
const sizeMap = useRef<Map<number, number>>(new Map());

const getSize = (index: number) => sizeMap.current.get(index) ?? 50;

const setSize = useCallback((index: number, size: number) => {
  sizeMap.current.set(index, size);
  listRef.current?.resetAfterIndex(index);
}, []);

// Each item must measure itself and call setSize
// Boilerplate vs @tanstack/react-virtual auto-handling
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`useVirtualizer` minimal misol:

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';
import { useRef } from 'react';

function ProductsList({ products }: { products: Product[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: products.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 80,
    overscan: 5,
  });

  return (
    <div
      ref={parentRef}
      style={{ height: 600, overflow: 'auto' }}
    >
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map((virtualItem) => (
          <div
            key={virtualItem.key}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              right: 0,
              height: `${virtualItem.size}px`,
              transform: `translateY(${virtualItem.start}px)`,
            }}
          >
            <ProductCard product={products[virtualItem.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

Variable heights with `measureElement`:

```tsx
function CommentsListVirtual({ comments }: { comments: Comment[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: comments.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 60,
    overscan: 3,
  });

  return (
    <div
      ref={parentRef}
      style={{ height: 500, overflow: 'auto' }}
    >
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map((virtualItem) => (
          <div
            key={virtualItem.key}
            data-index={virtualItem.index}
            ref={virtualizer.measureElement}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              right: 0,
              transform: `translateY(${virtualItem.start}px)`,
            }}
          >
            <CommentCard comment={comments[virtualItem.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

2D virtualization (grid):

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';

function PhotoGrid({ photos, columnCount = 4 }: {
  photos: Photo[];
  columnCount?: number;
}) {
  const parentRef = useRef<HTMLDivElement>(null);
  const rowCount = Math.ceil(photos.length / columnCount);

  const rowVirtualizer = useVirtualizer({
    count: rowCount,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 200,
    overscan: 2,
  });

  const columnVirtualizer = useVirtualizer({
    horizontal: true,
    count: columnCount,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 200,
    overscan: 1,
  });

  return (
    <div
      ref={parentRef}
      style={{ height: 600, width: 800, overflow: 'auto' }}
    >
      <div
        style={{
          height: `${rowVirtualizer.getTotalSize()}px`,
          width: `${columnVirtualizer.getTotalSize()}px`,
          position: 'relative',
        }}
      >
        {rowVirtualizer.getVirtualItems().map((virtualRow) =>
          columnVirtualizer.getVirtualItems().map((virtualColumn) => {
            const index = virtualRow.index * columnCount + virtualColumn.index;
            const photo = photos[index];
            if (!photo) return null;

            return (
              <div
                key={`${virtualRow.index}-${virtualColumn.index}`}
                style={{
                  position: 'absolute',
                  top: 0,
                  left: 0,
                  width: `${virtualColumn.size}px`,
                  height: `${virtualRow.size}px`,
                  transform: `translateX(${virtualColumn.start}px) translateY(${virtualRow.start}px)`,
                }}
              >
                <img src={photo.thumbnail} alt={photo.title} />
              </div>
            );
          })
        )}
      </div>
    </div>
  );
}
```

`scrollToIndex` API:

```tsx
function ChatList({ messages }: { messages: Message[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: messages.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 60,
  });

  const scrollToBottom = () => {
    virtualizer.scrollToIndex(messages.length - 1, { align: 'end' });
  };

  const scrollToMessage = (id: string) => {
    const index = messages.findIndex((m) => m.id === id);
    if (index !== -1) {
      virtualizer.scrollToIndex(index, { align: 'center' });
    }
  };

  return (
    <>
      <button onClick={scrollToBottom}>Oxiriga</button>
      <div ref={parentRef} style={{ height: 500, overflow: 'auto' }}>
        {/* ... virtualizer render ... */}
      </div>
    </>
  );
}
```

</details>

---

## Infinite Scroll + Virtualization

### Nazariya

Infinite scroll — foydalanuvchi pastga scroll qilganda keyingi sahifa ma'lumotlari avtomatik yuklanadi. Virtualization bilan birlashtirish — million item application'lar uchun standard pattern (Twitter feed, Instagram, e-commerce listing).

**Pattern:**

1. **Initial load** — birinchi 50-100 item.
2. **Scroll detection** — visible range list oxiriga yetganida.
3. **Fetch trigger** — keyingi page'ni server'dan olish.
4. **Append to list** — yangi items'ni mavjud array'ga qo'shish.
5. **Loading state** — pending yuklanish indikatori.

**Trigger mexanizm — 2 variant:**

1. **Library callback** — `onItemsRendered` (react-window) yoki `onChange` (TanStack Virtual):
   - Visible range last index list count'ga yaqin bo'lsa next page yuklanadi
   - Threshold: `endIndex >= itemCount - prefetchDistance`

2. **IntersectionObserver bottom sentinel** — list oxirida invisible sentinel element, viewport'ga kirsa next page trigger.

**TanStack Query integration:**

`useInfiniteQuery` hook va virtualization birlashishi natural fit. Pages'ni accumulator'ga aylantirish, `getNextPageParam` orqali pagination.

<details>
<summary><strong>Under the Hood</strong></summary>

**Race condition prevention:**

```typescript
// Without prevention: multiple concurrent fetches
async function loadMore() {
  // User scrolls → triggers loadMore 5 times in quick succession
  // → 5 concurrent fetches, response order undefined
}

// With prevention: in-flight flag
let isLoading = false;

async function loadMore() {
  if (isLoading) return;
  isLoading = true;

  try {
    const newPage = await fetchPage(nextCursor);
    setItems((prev) => [...prev, ...newPage.items]);
    setNextCursor(newPage.cursor);
  } finally {
    isLoading = false;
  }
}
```

TanStack Query `useInfiniteQuery` automatic deduplication.

**Cumulative pages flatten:**

```typescript
const data = useInfiniteQuery({
  queryKey: ['products'],
  queryFn: ({ pageParam }) => fetchProducts(pageParam),
  initialPageParam: 0,
  getNextPageParam: (lastPage) => lastPage.nextCursor,
});

// Pages structure:
// data.pages = [
//   { items: [...], nextCursor: '...' },
//   { items: [...], nextCursor: '...' },
//   ...
// ]

// Flatten for virtualizer:
const allItems = data.pages.flatMap((page) => page.items);
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`useVirtualizer` + `useInfiniteQuery`:

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';
import { useInfiniteQuery } from '@tanstack/react-query';
import { useEffect, useRef } from 'react';

interface ProductsPage {
  items: Product[];
  nextCursor: string | null;
}

async function fetchProductsPage(cursor: string | null): Promise<ProductsPage> {
  const url = `/api/products?cursor=${cursor ?? ''}&limit=50`;
  const response = await fetch(url);
  return response.json();
}

function InfiniteProductsList() {
  const parentRef = useRef<HTMLDivElement>(null);

  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
  } = useInfiniteQuery({
    queryKey: ['products'],
    queryFn: ({ pageParam }) => fetchProductsPage(pageParam),
    initialPageParam: null as string | null,
    getNextPageParam: (lastPage) => lastPage.nextCursor,
  });

  const allItems = data?.pages.flatMap((page) => page.items) ?? [];
  const totalCount = hasNextPage ? allItems.length + 1 : allItems.length;

  const virtualizer = useVirtualizer({
    count: totalCount,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 80,
    overscan: 5,
  });

  const virtualItems = virtualizer.getVirtualItems();
  const lastVisibleIndex = virtualItems.at(-1)?.index ?? -1;

  useEffect(() => {
    if (
      lastVisibleIndex >= allItems.length - 1 &&
      hasNextPage &&
      !isFetchingNextPage
    ) {
      fetchNextPage();
    }
  }, [hasNextPage, fetchNextPage, allItems.length, isFetchingNextPage, lastVisibleIndex]);

  if (isLoading) {
    return <div>Yuklanmoqda...</div>;
  }

  return (
    <div
      ref={parentRef}
      style={{ height: 600, overflow: 'auto' }}
    >
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          position: 'relative',
        }}
      >
        {virtualItems.map((virtualItem) => {
          const isLoaderRow = virtualItem.index >= allItems.length;
          const product = allItems[virtualItem.index];

          return (
            <div
              key={virtualItem.key}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                right: 0,
                height: `${virtualItem.size}px`,
                transform: `translateY(${virtualItem.start}px)`,
              }}
            >
              {isLoaderRow ? (
                hasNextPage ? <LoadingRow /> : <EndOfListRow />
              ) : (
                <ProductCard product={product} />
              )}
            </div>
          );
        })}
      </div>
    </div>
  );
}

function LoadingRow() {
  return (
    <div style={{ padding: 20, textAlign: 'center', color: '#888' }}>
      Yuklanmoqda...
    </div>
  );
}

function EndOfListRow() {
  return (
    <div style={{ padding: 20, textAlign: 'center', color: '#bbb' }}>
      Boshqa mahsulot yo'q.
    </div>
  );
}
```

`react-window` + `onItemsRendered`:

```tsx
import { FixedSizeList } from 'react-window';
import { useState, useEffect } from 'react';

function InfiniteListReactWindow() {
  const [items, setItems] = useState<Product[]>([]);
  const [hasMore, setHasMore] = useState(true);
  const [isLoading, setIsLoading] = useState(false);
  const [nextCursor, setNextCursor] = useState<string | null>(null);

  useEffect(() => {
    loadMore();
  }, []);

  const loadMore = async () => {
    if (isLoading || !hasMore) return;
    setIsLoading(true);

    try {
      const page = await fetchProductsPage(nextCursor);
      setItems((prev) => [...prev, ...page.items]);
      setNextCursor(page.nextCursor);
      setHasMore(page.nextCursor !== null);
    } finally {
      setIsLoading(false);
    }
  };

  const handleItemsRendered = ({
    visibleStopIndex,
  }: {
    visibleStopIndex: number;
  }) => {
    if (visibleStopIndex >= items.length - 5 && hasMore && !isLoading) {
      loadMore();
    }
  };

  return (
    <FixedSizeList
      height={600}
      width="100%"
      itemCount={items.length + (hasMore ? 1 : 0)}
      itemSize={80}
      onItemsRendered={handleItemsRendered}
    >
      {({ index, style }) => {
        if (index >= items.length) {
          return <div style={style}>Yuklanmoqda...</div>;
        }
        return (
          <div style={style}>
            <ProductCard product={items[index]} />
          </div>
        );
      }}
    </FixedSizeList>
  );
}
```

</details>

---

## Horizontal Virtualization va 2D Grid

### Nazariya

Horizontal virtualization — gorizontal scroll bilan ishlash. Use case'lar:

- **Image carousels** — 100+ photo'lar gorizontal scroll
- **Wide tables** — 50+ ustun bo'lgan jadval
- **Timeline scrubber** — uzun video timeline frame'lar
- **Calendar grid** — kun'lar gorizontal scroll

**2D grid virtualization** — rows va cols mustaqil virtualization. `@tanstack/react-virtual` rows va cols uchun alohida `useVirtualizer` ishlatadi. `react-window` `FixedSizeGrid`/`VariableSizeGrid` qayta birlashtirilgan API beradi.

**Layout considerations:**

- Container `display: flex; overflow-x: auto`
- Item `position: absolute; transform: translateX(offsetX)` (yoki `translateX + translateY` 2D'da)
- Scroll bar position'i UX uchun muhim (mobile'da yashirin)

<details>
<summary><strong>Kod Misollari</strong></summary>

Horizontal photo carousel (`react-window`):

```tsx
import { FixedSizeList } from 'react-window';

function PhotoCarousel({ photos }: { photos: Photo[] }) {
  return (
    <FixedSizeList
      layout="horizontal"
      height={200}
      width="100%"
      itemCount={photos.length}
      itemSize={250}
      overscanCount={3}
    >
      {({ index, style }) => (
        <div style={{ ...style, padding: 8 }}>
          <img
            src={photos[index].thumbnail}
            alt={photos[index].title}
            loading="lazy"
          />
        </div>
      )}
    </FixedSizeList>
  );
}
```

Wide table — horizontal + vertical virtualization (`@tanstack/react-virtual`):

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';
import { useRef } from 'react';

interface DataTable {
  columns: { id: string; label: string; width: number }[];
  rows: Record<string, string>[];
}

function VirtualizedTable({ columns, rows }: DataTable) {
  const parentRef = useRef<HTMLDivElement>(null);

  const rowVirtualizer = useVirtualizer({
    count: rows.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 40,
    overscan: 5,
  });

  const columnVirtualizer = useVirtualizer({
    horizontal: true,
    count: columns.length,
    getScrollElement: () => parentRef.current,
    estimateSize: (index) => columns[index].width,
    overscan: 2,
  });

  return (
    <div
      ref={parentRef}
      style={{ height: 600, width: '100%', overflow: 'auto' }}
    >
      <div
        style={{
          height: `${rowVirtualizer.getTotalSize()}px`,
          width: `${columnVirtualizer.getTotalSize()}px`,
          position: 'relative',
        }}
      >
        {rowVirtualizer.getVirtualItems().map((virtualRow) =>
          columnVirtualizer.getVirtualItems().map((virtualColumn) => {
            const row = rows[virtualRow.index];
            const column = columns[virtualColumn.index];
            const cellValue = row[column.id];

            return (
              <div
                key={`${virtualRow.key}-${virtualColumn.key}`}
                style={{
                  position: 'absolute',
                  top: 0,
                  left: 0,
                  width: `${virtualColumn.size}px`,
                  height: `${virtualRow.size}px`,
                  transform: `translateX(${virtualColumn.start}px) translateY(${virtualRow.start}px)`,
                  borderBottom: '1px solid #eee',
                  borderRight: '1px solid #eee',
                  padding: 8,
                  whiteSpace: 'nowrap',
                  overflow: 'hidden',
                  textOverflow: 'ellipsis',
                }}
              >
                {cellValue}
              </div>
            );
          })
        )}
      </div>
    </div>
  );
}
```

</details>

---

## Sticky Headers va Group Items

### Nazariya

Sticky headers — section header'lar viewport tepasida "yopishib" qoladigan pattern (iOS Contacts, alphabetical groups). Virtualization'da implement qilish murakkab — header item'lar boshqa item'lardan ajralib turishi va scroll position'iga qarab yangilanishi kerak.

**Pattern variantlari:**

1. **CSS `position: sticky`** — agar header ustun scroll context'da bo'lsa va virtualization absolute'da emas bo'lsa ishlaydi. Lekin transform context'da `position: sticky` ko'pincha buziladi.
2. **Manual sticky simulation** — virtualization render layer ichida absolute, header alohida fixed position bilan. Scroll listener current section'ni hisoblaydi.
3. **Group items virtualization** — har section header alohida virtual item, lekin sticky behavior alohida render layer'da.

**`@tanstack/react-virtual` `lanes` API** — masonry yoki multi-column virtualization (har item lane'ga ega).

<details>
<summary><strong>Kod Misollari</strong></summary>

Sticky headers manual simulation:

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';
import { useRef, useState, useEffect } from 'react';

interface ListItem {
  type: 'header' | 'item';
  letter?: string;
  contact?: Contact;
}

function ContactsListWithStickyHeaders({ contacts }: { contacts: Contact[] }) {
  const grouped = useMemo(() => {
    const groups = new Map<string, Contact[]>();
    for (const contact of contacts) {
      const letter = contact.name[0].toUpperCase();
      if (!groups.has(letter)) groups.set(letter, []);
      groups.get(letter)!.push(contact);
    }
    return Array.from(groups.entries())
      .sort(([a], [b]) => a.localeCompare(b))
      .flatMap(([letter, group]): ListItem[] => [
        { type: 'header', letter },
        ...group.map((contact): ListItem => ({ type: 'item', contact })),
      ]);
  }, [contacts]);

  const parentRef = useRef<HTMLDivElement>(null);
  const [activeStickyHeader, setActiveStickyHeader] = useState<string | null>(null);

  const virtualizer = useVirtualizer({
    count: grouped.length,
    getScrollElement: () => parentRef.current,
    estimateSize: (index) => (grouped[index].type === 'header' ? 32 : 60),
    overscan: 5,
  });

  // Visible range'ning birinchi va oxirgi index'larini extract qilamiz —
  // shu primitive qiymatlar deps array uchun stable. `virtualizer.getVirtualItems()`
  // har render'da yangi array qaytaradi, demak unstable dep'sifatida ishlatilmaydi.
  const visibleItems = virtualizer.getVirtualItems();
  const firstVisibleIndex = visibleItems[0]?.index ?? -1;
  const lastVisibleIndex = visibleItems.at(-1)?.index ?? -1;

  useEffect(() => {
    if (firstVisibleIndex < 0) return;

    let activeHeader: string | null = null;
    for (let i = firstVisibleIndex; i <= lastVisibleIndex && i < grouped.length; i++) {
      const item = grouped[i];
      if (item.type === 'header') {
        activeHeader = item.letter ?? null;
      }
    }

    setActiveStickyHeader(activeHeader);
  }, [firstVisibleIndex, lastVisibleIndex, grouped]);

  return (
    <div
      ref={parentRef}
      style={{ height: 600, overflow: 'auto', position: 'relative' }}
    >
      {activeStickyHeader && (
        <div
          style={{
            position: 'sticky',
            top: 0,
            zIndex: 10,
            background: 'rgba(255, 255, 255, 0.95)',
            padding: '8px 12px',
            fontWeight: 'bold',
            borderBottom: '1px solid #ddd',
          }}
        >
          {activeStickyHeader}
        </div>
      )}

      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map((virtualItem) => {
          const item = grouped[virtualItem.index];

          return (
            <div
              key={virtualItem.key}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                right: 0,
                height: `${virtualItem.size}px`,
                transform: `translateY(${virtualItem.start}px)`,
              }}
            >
              {item.type === 'header' ? (
                <SectionHeader letter={item.letter!} />
              ) : (
                <ContactRow contact={item.contact!} />
              )}
            </div>
          );
        })}
      </div>
    </div>
  );
}

function SectionHeader({ letter }: { letter: string }) {
  return (
    <div style={{ padding: '8px 12px', background: '#f5f5f5', fontWeight: 'bold' }}>
      {letter}
    </div>
  );
}

function ContactRow({ contact }: { contact: Contact }) {
  return (
    <div style={{ padding: 12, borderBottom: '1px solid #eee' }}>
      {contact.name}
    </div>
  );
}
```

</details>

---

## Use Cases

### Nazariya

Virtualization keng tarqalgan use case'lari:

**1. Long lists (1000+ items)** — eng tipik holat. Product catalogs, search results, user lists.

**2. Tables** — finance dashboards, admin panels, data analysis tools. Wide va tall jadvallar 2D virtualization talab qiladi.

**3. Chat/Messaging** — Telegram, Slack, Discord stilidagi message list'lar. Variable heights (text uzunligi har xil), `scrollToIndex` API yangi xabar uchun, sticky date headers.

**4. Dropdown/Combobox** — autocomplete katta dataset (countries, cities, products). 1000+ option'lar.

**5. Calendar/Timeline** — yil bo'yi events, video editor timeline frame'lar. Horizontal va vertical kombinatsiya.

**6. Photo Galleries** — 2D grid, masonry layout, lazy loaded thumbnails.

**7. Code Editors** — Monaco/CodeMirror file lines virtualization (lekin bu library'lar custom virtualization ishlatadi, react-window'siz).

**8. Tree Views** — file explorers, org charts. Flatten hierarchy + indent depth orqali render.

<details>
<summary><strong>Kod Misollari</strong></summary>

Chat list with `scrollToIndex` on new message:

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';
import { useEffect, useRef } from 'react';

interface ChatMessage {
  id: string;
  author: string;
  text: string;
  timestamp: number;
}

function ChatRoom({ messages }: { messages: ChatMessage[] }) {
  const parentRef = useRef<HTMLDivElement>(null);
  const lastMessageCountRef = useRef(messages.length);

  const virtualizer = useVirtualizer({
    count: messages.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 60,
    overscan: 5,
    paddingEnd: 16,
  });

  useEffect(() => {
    if (messages.length > lastMessageCountRef.current) {
      virtualizer.scrollToIndex(messages.length - 1, { align: 'end' });
    }
    lastMessageCountRef.current = messages.length;
  }, [messages.length, virtualizer]);

  return (
    <div
      ref={parentRef}
      style={{ height: 500, overflow: 'auto' }}
    >
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map((virtualItem) => {
          const message = messages[virtualItem.index];

          return (
            <div
              key={message.id}
              data-index={virtualItem.index}
              ref={virtualizer.measureElement}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                right: 0,
                transform: `translateY(${virtualItem.start}px)`,
              }}
            >
              <MessageBubble message={message} />
            </div>
          );
        })}
      </div>
    </div>
  );
}

function MessageBubble({ message }: { message: ChatMessage }) {
  return (
    <div style={{ padding: '8px 12px' }}>
      <strong>{message.author}</strong>
      <p style={{ margin: '4px 0', whiteSpace: 'pre-wrap' }}>{message.text}</p>
      <small style={{ color: '#888' }}>
        {new Date(message.timestamp).toLocaleTimeString()}
      </small>
    </div>
  );
}
```

Virtualized combobox (autocomplete):

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';
import { useState, useRef, useMemo } from 'react';

interface ComboboxProps {
  options: string[];
  value: string;
  onChange: (value: string) => void;
}

function VirtualizedCombobox({ options, value, onChange }: ComboboxProps) {
  const [search, setSearch] = useState('');
  const [isOpen, setIsOpen] = useState(false);
  const parentRef = useRef<HTMLDivElement>(null);

  const filtered = useMemo(
    () => options.filter((opt) => opt.toLowerCase().includes(search.toLowerCase())),
    [options, search]
  );

  const virtualizer = useVirtualizer({
    count: filtered.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 32,
    overscan: 5,
  });

  return (
    <div style={{ position: 'relative', width: 300 }}>
      <input
        value={search || value}
        onChange={(e) => {
          setSearch(e.target.value);
          setIsOpen(true);
        }}
        onFocus={() => setIsOpen(true)}
        placeholder="Tanlang..."
      />

      {isOpen && (
        <div
          ref={parentRef}
          style={{
            position: 'absolute',
            top: '100%',
            left: 0,
            right: 0,
            maxHeight: 240,
            overflow: 'auto',
            background: 'white',
            border: '1px solid #ccc',
            borderRadius: 4,
            zIndex: 100,
          }}
        >
          <div
            style={{
              height: `${virtualizer.getTotalSize()}px`,
              position: 'relative',
            }}
          >
            {virtualizer.getVirtualItems().map((virtualItem) => {
              const option = filtered[virtualItem.index];

              return (
                <div
                  key={virtualItem.key}
                  onClick={() => {
                    onChange(option);
                    setSearch('');
                    setIsOpen(false);
                  }}
                  style={{
                    position: 'absolute',
                    top: 0,
                    left: 0,
                    right: 0,
                    height: `${virtualItem.size}px`,
                    transform: `translateY(${virtualItem.start}px)`,
                    padding: '8px 12px',
                    cursor: 'pointer',
                    background: option === value ? '#e0f0ff' : 'white',
                  }}
                >
                  {option}
                </div>
              );
            })}
          </div>
        </div>
      )}
    </div>
  );
}
```

</details>

---

## Performance Comparison

### Nazariya

Virtualization performance ta'siri item count'ga proportional. Real measurement workflow (cross-ref `34-profiling.md`):

1. **Build production mode** — DevTools production build profile.
2. **Initial render** — recordings boshlanishi, item count parameterized.
3. **Scroll FPS** — Chrome DevTools Performance tab, scroll user interaction simulation.
4. **Memory** — Chrome DevTools Memory tab, snapshot before va after.

**Trade-off boundaries:**

- **< 100 items:** Virtualization overhead > naive render overhead. Naive afzal.
- **100-1000 items:** Border zone — komponent murakkabligi muhim. Heavy items (rich content, charts) virtualization foydali, simple items (text) naive ham OK.
- **> 1000 items:** Virtualization majburiy.

> **Real measurements item complexity'ga bog'liq.** ProductCard 5 KB JSX, ChartItem 50 KB JSX (recharts) — initial render vaqti differ.

<details>
<summary><strong>Kod Misollari</strong></summary>

Profile setup with `<Profiler>`:

```tsx
import { Profiler, type ProfilerOnRenderCallback } from 'react';

const onRender: ProfilerOnRenderCallback = (
  id,
  phase,
  actualDuration,
  baseDuration
) => {
  console.log({
    id,
    phase,
    actualDuration: `${actualDuration.toFixed(2)}ms`,
    baseDuration: `${baseDuration.toFixed(2)}ms`,
    efficiency: `${((1 - actualDuration / baseDuration) * 100).toFixed(0)}%`,
  });
};

function App() {
  return (
    <>
      <Profiler id="naive-list" onRender={onRender}>
        <NaiveProductList products={products} />
      </Profiler>

      <Profiler id="virtualized-list" onRender={onRender}>
        <VirtualizedProductList products={products} />
      </Profiler>
    </>
  );
}
```

Benchmark utility:

```tsx
import { useState, useEffect } from 'react';

function useBenchmark(label: string, dependency: unknown) {
  useEffect(() => {
    const start = performance.now();

    requestAnimationFrame(() => {
      const duration = performance.now() - start;
      console.log(`[${label}] Render time: ${duration.toFixed(2)}ms`);
    });
  }, [dependency, label]);
}

function BenchmarkPage() {
  const [count, setCount] = useState(100);
  const products = useMemo(() => generateProducts(count), [count]);

  useBenchmark(`Naive (${count} items)`, products);

  return (
    <>
      <input
        type="range"
        min={100}
        max={10000}
        value={count}
        onChange={(e) => setCount(Number(e.target.value))}
      />
      <p>{count} items</p>
      <NaiveProductList products={products} />
    </>
  );
}
```

</details>

---

## Edge Cases va Gotchas

### Strict Mode 2x render impact

R18+ Strict Mode dev'da `useState` setter va `useEffect` 2x sodir bo'ladi. Virtualization scroll handler'da:

- `setScrollTop` 2x → calculate range 2x → re-render 2x
- ResizeObserver callback 2x → `setHeights` 2x

Bu development-only behavior, lekin profiling natijalari noto'g'ri ko'rsatadi. Production build profile (cross-ref `34-profiling.md`) yoki Strict Mode olib tashlash development'da.

### `key={index}` xato

Virtualization'da `key={index}` ishlatilsa — har scroll'da indices o'zgaradi va React Reconciler item'larni unmount/mount qiladi. State lost (controlled inputs, local UI state), Fiber rebuild, performance regression. Item'ning unique ID'si (`product.id`, `message.id`) ishlatish shart (cross-ref `08-list-rendering.md`).

```tsx
// ❌ NOTO'G'RI
<div key={index}>{...}</div>

// ✅ TO'G'RI
<div key={product.id}>{...}</div>
```

### Variable heights jumping scrollbar

Variable heights virtualization'da estimate'dan haqiqiy size farq bo'lsa — scroll bar position jumpy ko'rinadi. Yechim'lar:

- `estimateSize` ni real average'ga yaqin qilish
- `measureElement` har item uchun ulash (TanStack Virtual)
- Items pre-measure (server-side render qilingan height)

### Anchor on resize

Window resize bo'lsa container height o'zgaradi → visible range o'zgaradi → user "lost in scroll". Pattern: scroll position'ni "anchor" item'ga bog'lash:

```tsx
useEffect(() => {
  const handleResize = () => {
    const visible = virtualizer.getVirtualItems();
    if (visible.length > 0) {
      virtualizer.scrollToIndex(visible[0].index, { align: 'start' });
    }
  };
  window.addEventListener('resize', handleResize);
  return () => window.removeEventListener('resize', handleResize);
}, [virtualizer]);
```

### Accessibility — Screen reader Visible Items'ni "list end" deb ko'radi

Virtualization fundamental a11y muammosi: faqat visible items DOM'da → screen reader (NVDA, JAWS, VoiceOver) butun list'ni emas, faqat visible portion'ni ko'radi. "10000 items" da'vo qilingan list 12 ta visible item bilan navigatsiya qilinsa, user "list end"'ga yetdim deb noto'g'ri yo'l oladi.

**ARIA roles + size attributes — standart pattern:**

```tsx
// ❌ Native list — screen reader 12 visible item'ni "list end" deb e'lon qiladi
<div ref={parentRef} style={{ height: 600, overflow: 'auto' }}>
  <div style={{ height: virtualizer.getTotalSize() }}>
    {virtualizer.getVirtualItems().map(virtualItem => (
      <div key={virtualItem.key}>...</div>
    ))}
  </div>
</div>

// ✅ ARIA-aware virtualization
<div
  ref={parentRef}
  role="list"
  aria-label="Mahsulotlar ro'yxati"
  aria-rowcount={items.length}  // ← TOTAL count (visible emas)
  style={{ height: 600, overflow: 'auto' }}
>
  <div style={{ height: virtualizer.getTotalSize() }}>
    {virtualizer.getVirtualItems().map(virtualItem => (
      <div
        key={virtualItem.key}
        role="listitem"
        aria-rowindex={virtualItem.index + 1}  // ← 1-based, real position
        aria-setsize={items.length}              // ← total set size
        aria-posinset={virtualItem.index + 1}    // ← position in set
      >
        {/* content */}
      </div>
    ))}
  </div>
</div>
```

**ARIA atributlari semantikasi:**

| Atribut | Maqsad | Misol qiymati |
|---------|--------|---------------|
| `aria-rowcount` | Total rows (grid/table'da) | `10000` |
| `aria-rowindex` | Joriy item position (1-based) | `4523` |
| `aria-setsize` | Total set size (listitem'da) | `10000` |
| `aria-posinset` | Position in set (1-based) | `4523` |

**Keyboard navigation challenges:**

- **Tab/Shift+Tab** — visible items orasidagi focus normal, lekin scroll oxiriga yetganda navigatsiya uziladi
- **Arrow keys (Up/Down)** — custom keyboard handler kerak: focus next item, agar visible bo'lmasa `scrollToIndex` chaqirish
- **Page Up/Down, Home/End** — full container scroll uchun custom handler

**Modern alternative — CSS `content-visibility: auto`:**

Native browser virtualization-like behavior. Browser hidden elements'ni render skip qiladi (lekin DOM'da qoladi, screen reader uchun mavjud). Modern, lekin variable heights uchun manual `contain-intrinsic-size` kerak. Bundle size 0 (CSS feature).

```css
.list-item {
  content-visibility: auto;
  contain-intrinsic-size: auto 60px; /* estimate height */
}
```

Browser support: Chrome 85+ (2020-08), Firefox 125+ (2024-04), Safari 18+ (2024-09). Cross-browser parity 2024 yil oxiridan boshlangan.

### Programmatic scroll racing

`scrollToIndex(i)` chaqirilgandan so'ng darhol component height o'zgartirilsa — measurement va scroll position race condition'ga tushadi. TanStack Virtual `scrollToIndex(index, options?): void` **synchronous** API — qiymat qaytarmaydi, lekin scroll natija ResizeObserver measurement bilan asynchronous to'g'irlanadi (item haqiqiy size estimateSize'dan farq qilsa). Shu sababli `await virtualizer.scrollToIndex(...)` qiymat bermaydi (`undefined` await qiladi).

Ketma-ket scroll'lar yoki measurement settle bo'lguncha kutish uchun pattern:

```tsx
virtualizer.scrollToIndex(targetIndex, { align: 'center' });

// Layout settle uchun rAF kutish, keyin ikkinchi scroll (measurement
// haqiqiy heights bilan adjust qildi)
requestAnimationFrame(() => {
  requestAnimationFrame(() => {
    // Endi accurate position'da
    virtualizer.scrollToIndex(targetIndex, { align: 'center' });
  });
});
```

`react-window` `scrollToItem` ham sync (void qaytaradi); har ikkala library'da ResizeObserver callbacks browser pipeline bilan async — variable heights case'da scroll position'i layout'dan keyin to'g'irlanadi.

---

## Common Mistakes

### ❌ Xato 1: `key={index}` ishlatish

```tsx
// ❌ State lost on scroll
{virtualizer.getVirtualItems().map((virtualItem) => (
  <div key={virtualItem.index}>...</div>
))}

// ✅ TO'G'RI — stable key (item ID)
{virtualizer.getVirtualItems().map((virtualItem) => {
  const item = items[virtualItem.index];
  return <div key={item.id}>...</div>;
})}

// virtualItem.key TanStack Virtual default'da `index` — `key={virtualItem.key}`
// stable EMAS (xuddi `key={index}` kabi). Stable key uchun `useVirtualizer`'ga
// `getItemKey` callback uzatish:
const virtualizer = useVirtualizer({
  count: items.length,
  getScrollElement: () => parentRef.current,
  estimateSize: () => 60,
  getItemKey: (index) => items[index].id, // ← stable item ID
});

// Endi virtualItem.key = items[index].id (stable):
{virtualizer.getVirtualItems().map((virtualItem) => (
  <div key={virtualItem.key}>...</div>
))}
```

### ❌ Xato 2: `style` prop'ni skip qilish (`react-window`)

```tsx
// ❌ Item position'ni saqlamaydi → barcha item'lar bir joyda overlap
function Row({ index, style, data }: RowProps) {
  return <div>{data[index].name}</div>;
}

// ✅ TO'G'RI — style'ni element'ga uzatish
function Row({ index, style, data }: RowProps) {
  return <div style={style}>{data[index].name}</div>;
}
```

### ❌ Xato 3: Container'ga `height` bermaslik

```tsx
// ❌ Virtualizer scroll element height'ni topa olmaydi → 0 visible items
<div ref={parentRef} style={{ overflow: 'auto' }}>
  {/* virtualizer render */}
</div>

// ✅ TO'G'RI — explicit height yoki flex parent
<div ref={parentRef} style={{ height: 600, overflow: 'auto' }}>
  {/* virtualizer render */}
</div>
```

### ❌ Xato 4: `estimateSize` to'g'ri kelmaslik

```tsx
// ❌ Items haqiqiy 60px, estimate 100 — scrollbar jumpy
useVirtualizer({
  count: items.length,
  estimateSize: () => 100,
});

// ✅ TO'G'RI — real average bilan
useVirtualizer({
  count: items.length,
  estimateSize: () => 60,
});
```

### ❌ Xato 5: Heavy work scroll handler'da

```tsx
// ❌ Har scroll event'da heavy computation → scroll lag
const handleScroll = (e: React.UIEvent<HTMLDivElement>) => {
  setScrollTop(e.currentTarget.scrollTop);
  expensiveAnalyticsTrack(e); // har frame chaqirilmasin
};

// ✅ TO'G'RI — debounce yoki rAF
const handleScrollDebounced = useMemo(
  () => debounce((scrollTop: number) => {
    expensiveAnalyticsTrack(scrollTop);
  }, 200),
  []
);

const handleScroll = (e: React.UIEvent<HTMLDivElement>) => {
  setScrollTop(e.currentTarget.scrollTop);
  handleScrollDebounced(e.currentTarget.scrollTop);
};
```

---

## Amaliy Mashqlar

### Mashq 1: Pure React Fixed-Height Virtualization (Oson)

Pure React ishlatib `VirtualList<T>` komponent yarating: fixed item height, generic typed items, render prop pattern. Container height va overscan props.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useState, useRef, type ReactNode, type UIEvent } from 'react';

interface VirtualListProps<T> {
  items: T[];
  itemHeight: number;
  height: number;
  overscan?: number;
  renderItem: (item: T, index: number) => ReactNode;
  getItemKey: (item: T, index: number) => string | number;
}

function VirtualList<T>({
  items,
  itemHeight,
  height,
  overscan = 3,
  renderItem,
  getItemKey,
}: VirtualListProps<T>) {
  const [scrollTop, setScrollTop] = useState(0);
  const containerRef = useRef<HTMLDivElement>(null);

  const totalHeight = items.length * itemHeight;
  const visibleCount = Math.ceil(height / itemHeight);
  const startIndex = Math.max(0, Math.floor(scrollTop / itemHeight) - overscan);
  const endIndex = Math.min(items.length, startIndex + visibleCount + overscan * 2);
  const offsetY = startIndex * itemHeight;

  const handleScroll = (e: UIEvent<HTMLDivElement>) => {
    setScrollTop(e.currentTarget.scrollTop);
  };

  return (
    <div
      ref={containerRef}
      onScroll={handleScroll}
      style={{ height, overflowY: 'auto', position: 'relative' }}
    >
      <div style={{ height: totalHeight, position: 'relative' }}>
        <div
          style={{
            position: 'absolute',
            top: 0,
            left: 0,
            right: 0,
            transform: `translateY(${offsetY}px)`,
          }}
        >
          {items.slice(startIndex, endIndex).map((item, i) => (
            <div
              key={getItemKey(item, startIndex + i)}
              style={{ height: itemHeight }}
            >
              {renderItem(item, startIndex + i)}
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}
```

**Tushuntirish:**

- Generic `<T>` har qanday item type qabul qiladi (Product, Comment, User).
- `getItemKey` — stable key callback (`product.id`).
- Spacer pattern (`totalHeight`) scroll bar to'g'ri proporsiya.
- `transform: translateY` GPU-accelerated.
- Overscan (default 3) smooth scroll uchun.

</details>

---

### Mashq 2: `react-window` FixedSizeList Bilan Product List (Oson)

`react-window` `FixedSizeList`'ni ishlatib 1000+ Product'ni virtualization qiling. `React.memo` + `areEqual` per-item.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { FixedSizeList, areEqual, type ListChildComponentProps } from 'react-window';
import { memo } from 'react';

interface Product {
  id: string;
  name: string;
  price: number;
  imageUrl: string;
}

interface RowProps extends ListChildComponentProps {
  data: Product[];
}

const Row = memo(({ index, style, data }: RowProps) => {
  const product = data[index];

  return (
    <div
      style={{
        ...style,
        display: 'flex',
        alignItems: 'center',
        padding: 12,
        borderBottom: '1px solid #eee',
      }}
    >
      <img
        src={product.imageUrl}
        alt={product.name}
        style={{ width: 48, height: 48, marginRight: 12 }}
      />
      <div>
        <h4 style={{ margin: 0 }}>{product.name}</h4>
        <p style={{ margin: 0, color: '#888' }}>${product.price.toFixed(2)}</p>
      </div>
    </div>
  );
}, areEqual);

function ProductsList({ products }: { products: Product[] }) {
  return (
    <FixedSizeList
      height={600}
      width="100%"
      itemCount={products.length}
      itemSize={72}
      itemData={products}
      overscanCount={5}
    >
      {Row}
    </FixedSizeList>
  );
}
```

**Tushuntirish:**

- `Row` komponent `memo` + `areEqual` bilan wrap qilingan — props o'zgarmasa skip.
- `style` prop majburiy — library `transform`'ga aylantiriladi.
- `itemData={products}` — Row komponent'iga `data` prop sifatida uzatiladi.
- `overscanCount={5}` — visible range tashqarida 5 ta qo'shimcha item.

</details>

---

### Mashq 3: Variable Heights bilan Comments List (O'rta)

`@tanstack/react-virtual` ishlatib variable height comment list yarating. Har comment'ning haqiqiy height'i text uzunligiga bog'liq. `measureElement` automatic measurement.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';
import { useRef } from 'react';

interface Comment {
  id: string;
  author: string;
  text: string;
  createdAt: string;
}

function CommentsList({ comments }: { comments: Comment[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: comments.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 80,
    overscan: 5,
  });

  return (
    <div
      ref={parentRef}
      style={{ height: 500, overflow: 'auto' }}
    >
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map((virtualItem) => {
          const comment = comments[virtualItem.index];

          return (
            <div
              key={comment.id}
              data-index={virtualItem.index}
              ref={virtualizer.measureElement}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                right: 0,
                transform: `translateY(${virtualItem.start}px)`,
              }}
            >
              <CommentCard comment={comment} />
            </div>
          );
        })}
      </div>
    </div>
  );
}

function CommentCard({ comment }: { comment: Comment }) {
  return (
    <div style={{ padding: 12, borderBottom: '1px solid #eee' }}>
      <div style={{ display: 'flex', justifyContent: 'space-between', marginBottom: 4 }}>
        <strong>{comment.author}</strong>
        <small style={{ color: '#888' }}>
          {new Date(comment.createdAt).toLocaleDateString()}
        </small>
      </div>
      <p style={{ margin: 0, whiteSpace: 'pre-wrap' }}>{comment.text}</p>
    </div>
  );
}
```

**Tushuntirish:**

- `estimateSize: () => 80` — initial taxminiy height (real comment uchun average).
- `ref={virtualizer.measureElement}` — ResizeObserver attach, automatic real height measurement.
- `data-index` — library tomonidan kuzatiladigan attribute.
- Subsequent scroll'larda haqiqiy heights cache'dan ishlatiladi.
- Rasmiy `@tanstack/react-virtual` pattern: `position: absolute; top: 0; left: 0; right: 0` (full-width container'da) + `transform: translateY(${virtualItem.start}px)` GPU-accelerated positioning uchun. Bitta render layer ichida barcha visible items absolute, container `position: relative`.

</details>

---

### Mashq 4: Infinite Scroll + Virtualization + TanStack Query (O'rta)

`useInfiniteQuery` va `useVirtualizer` birlashtirib infinite scroll product list yarating. Loading state, end of list state, race condition prevention.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useInfiniteQuery } from '@tanstack/react-query';
import { useVirtualizer } from '@tanstack/react-virtual';
import { useEffect, useRef } from 'react';

interface ProductsPage {
  items: Product[];
  nextCursor: string | null;
}

async function fetchProducts(cursor: string | null): Promise<ProductsPage> {
  const url = `/api/products?cursor=${cursor ?? ''}&limit=50`;
  const response = await fetch(url);
  if (!response.ok) throw new Error('Network error');
  return response.json();
}

function InfiniteProductsList() {
  const parentRef = useRef<HTMLDivElement>(null);

  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
    isError,
  } = useInfiniteQuery({
    queryKey: ['products'],
    queryFn: ({ pageParam }) => fetchProducts(pageParam),
    initialPageParam: null as string | null,
    getNextPageParam: (lastPage) => lastPage.nextCursor,
  });

  const allItems = data?.pages.flatMap((page) => page.items) ?? [];
  const totalCount = hasNextPage ? allItems.length + 1 : allItems.length;

  const virtualizer = useVirtualizer({
    count: totalCount,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 80,
    overscan: 5,
  });

  const virtualItems = virtualizer.getVirtualItems();
  const lastVisibleIndex = virtualItems.at(-1)?.index ?? -1;

  useEffect(() => {
    if (
      lastVisibleIndex >= allItems.length - 1 &&
      hasNextPage &&
      !isFetchingNextPage
    ) {
      fetchNextPage();
    }
  }, [hasNextPage, fetchNextPage, allItems.length, isFetchingNextPage, lastVisibleIndex]);

  if (isLoading) return <PageSkeleton />;
  if (isError) return <ErrorState onRetry={() => fetchNextPage()} />;

  return (
    <div
      ref={parentRef}
      style={{ height: 600, overflow: 'auto' }}
    >
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          position: 'relative',
        }}
      >
        {virtualItems.map((virtualItem) => {
          const isLoaderRow = virtualItem.index >= allItems.length;
          const product = allItems[virtualItem.index];

          return (
            <div
              key={isLoaderRow ? 'loader' : product.id}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                right: 0,
                height: `${virtualItem.size}px`,
                transform: `translateY(${virtualItem.start}px)`,
              }}
            >
              {isLoaderRow ? (
                hasNextPage ? <LoaderRow /> : <EndRow />
              ) : (
                <ProductCard product={product} />
              )}
            </div>
          );
        })}
      </div>
    </div>
  );
}

function LoaderRow() {
  return (
    <div style={{ padding: 16, textAlign: 'center', color: '#888' }}>
      Yuklanmoqda...
    </div>
  );
}

function EndRow() {
  return (
    <div style={{ padding: 16, textAlign: 'center', color: '#bbb' }}>
      Boshqa mahsulot yo'q.
    </div>
  );
}
```

**Tushuntirish:**

- `useInfiniteQuery` automatic deduplication (race condition prevention).
- `count: totalCount` — visible items + loader row (hasNextPage ? +1).
- `useEffect` har visible range update'ida last item index'ni tekshiradi.
- Threshold: `lastItem.index >= allItems.length - 1` — list'ning oxiriga yetganida fetch.
- Loader row va end row alohida visual states.

</details>

---

### Mashq 5: Production Chat Application (Qiyin)

To'liq production-grade chat list yarating: variable heights (text uzunligiga ko'ra), `scrollToBottom` yangi xabar uchun, sticky date headers, `measureElement` automatic measurement, optimistic UI yangi message uchun, infinite scroll history yuklash uchun.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useVirtualizer } from '@tanstack/react-virtual';
import { useEffect, useMemo, useRef, useState } from 'react';

interface ChatMessage {
  id: string;
  author: string;
  text: string;
  timestamp: number;
  status: 'sent' | 'pending' | 'failed';
}

interface MessagesPage {
  items: ChatMessage[];
  nextCursor: string | null;
}

interface ListItem {
  type: 'date' | 'message';
  date?: string;
  message?: ChatMessage;
  key: string;
}

function ChatRoom({
  messages,
  onLoadMore,
  hasMore,
  isLoadingMore,
}: {
  messages: ChatMessage[];
  onLoadMore: () => void;
  hasMore: boolean;
  isLoadingMore: boolean;
}) {
  const parentRef = useRef<HTMLDivElement>(null);
  const lastMessageCountRef = useRef(messages.length);
  const [activeStickyDate, setActiveStickyDate] = useState<string | null>(null);

  const flattenedItems = useMemo<ListItem[]>(() => {
    const result: ListItem[] = [];
    let lastDate = '';

    if (hasMore) {
      result.push({ type: 'date', date: 'Yuklanmoqda...', key: 'loader' });
    }

    for (const message of messages) {
      const date = new Date(message.timestamp).toLocaleDateString();
      if (date !== lastDate) {
        result.push({ type: 'date', date, key: `date-${date}` });
        lastDate = date;
      }
      result.push({ type: 'message', message, key: message.id });
    }

    return result;
  }, [messages, hasMore]);

  const virtualizer = useVirtualizer({
    count: flattenedItems.length,
    getScrollElement: () => parentRef.current,
    estimateSize: (index) => (flattenedItems[index].type === 'date' ? 32 : 60),
    overscan: 5,
  });

  useEffect(() => {
    if (messages.length > lastMessageCountRef.current) {
      requestAnimationFrame(() => {
        virtualizer.scrollToIndex(flattenedItems.length - 1, { align: 'end' });
      });
    }
    lastMessageCountRef.current = messages.length;
  }, [messages.length, flattenedItems.length, virtualizer]);

  useEffect(() => {
    const visible = virtualizer.getVirtualItems();
    if (visible.length === 0) return;

    if (visible[0].index <= 1 && hasMore && !isLoadingMore) {
      onLoadMore();
    }

    let activeDate: string | null = null;
    for (const item of visible) {
      const flat = flattenedItems[item.index];
      if (flat.type === 'date' && flat.date !== 'Yuklanmoqda...') {
        activeDate = flat.date ?? null;
      }
    }
    setActiveStickyDate(activeDate);
  }, [virtualizer, flattenedItems, hasMore, isLoadingMore, onLoadMore]);

  return (
    <div
      style={{
        position: 'relative',
        height: 500,
        border: '1px solid #ddd',
        borderRadius: 4,
      }}
    >
      {activeStickyDate && (
        <div
          style={{
            position: 'absolute',
            top: 8,
            left: '50%',
            transform: 'translateX(-50%)',
            zIndex: 10,
            background: 'rgba(255, 255, 255, 0.95)',
            border: '1px solid #ddd',
            borderRadius: 16,
            padding: '4px 12px',
            fontSize: 12,
            fontWeight: 'bold',
            color: '#666',
          }}
        >
          {activeStickyDate}
        </div>
      )}

      <div
        ref={parentRef}
        style={{ height: '100%', overflow: 'auto' }}
      >
        <div
          style={{
            height: `${virtualizer.getTotalSize()}px`,
            position: 'relative',
          }}
        >
          {virtualizer.getVirtualItems().map((virtualItem) => {
            const item = flattenedItems[virtualItem.index];

            return (
              <div
                key={item.key}
                data-index={virtualItem.index}
                ref={virtualizer.measureElement}
                style={{
                  position: 'absolute',
                  top: 0,
                  left: 0,
                  right: 0,
                  transform: `translateY(${virtualItem.start}px)`,
                }}
              >
                {item.type === 'date' ? (
                  <DateBadge label={item.date!} />
                ) : (
                  <MessageBubble message={item.message!} />
                )}
              </div>
            );
          })}
        </div>
      </div>
    </div>
  );
}

function DateBadge({ label }: { label: string }) {
  return (
    <div
      style={{
        padding: '8px 12px',
        textAlign: 'center',
        color: '#888',
        fontSize: 12,
      }}
    >
      {label}
    </div>
  );
}

function MessageBubble({ message }: { message: ChatMessage }) {
  return (
    <div
      style={{
        padding: '8px 12px',
        opacity: message.status === 'pending' ? 0.6 : 1,
      }}
    >
      <div style={{ display: 'flex', justifyContent: 'space-between' }}>
        <strong>{message.author}</strong>
        <small style={{ color: '#888' }}>
          {new Date(message.timestamp).toLocaleTimeString()}
        </small>
      </div>
      <p style={{ margin: '4px 0', whiteSpace: 'pre-wrap' }}>{message.text}</p>
      {message.status === 'failed' && (
        <span style={{ color: 'crimson', fontSize: 12 }}>Yuborilmadi</span>
      )}
    </div>
  );
}
```

**Tushuntirish:**

- **Variable heights** — `measureElement` automatic measurement, message text uzunligiga moslashadi.
- **`scrollToBottom`** — `useEffect` yangi message detect qilib `scrollToIndex` chaqiradi (rAF frame bilan, layout settle bo'lguncha).
- **Sticky date headers** — visible range scan, `activeStickyDate` state'da.
- **Infinite history loading** — visible range tepasiga yetganda `onLoadMore` trigger.
- **Optimistic UI** — `status: 'pending'` xabar opacity'si pasaytiriladi, `failed` indicator alohida.
- **Race condition prevention** — `lastMessageCountRef` count'ni track qiladi, scrollToIndex faqat yangi xabarda.

</details>

---

## Xulosa

- **Virtualization** — uzun ro'yxatlar (10,000+ items) va katta jadvallarni render qilish uchun fundamental texnika. DOM'da faqat ko'rinadigan element'lar saqlanadi, qolganlari virtual offset orqali joylashtiriladi.
- **Windowing algorithm** — 4 parametr bilan ishlaydi: `scrollTop`, `itemHeight`, `containerHeight`, `itemCount`. Visible range formula `Math.floor(scrollTop / itemHeight)` dan `Math.ceil((scrollTop + height) / itemHeight)` gacha. Overscan smooth scroll uchun.
- **Spacer pattern** — total height saqlash uchun container ichida `itemCount × itemHeight` balandlikdagi spacer. Render layer `transform: translateY(offsetY)` GPU-accelerated.
- **Pure React implementation** — `useRef` + `useState(scrollTop)` + visible range calc + `requestAnimationFrame` throttle. Generic typing va render prop pattern.
- **Variable heights** — pre-known heights array yoki estimate-then-measure pattern. `ResizeObserver` automatic measurement, offset cache `Map<index, height>` yangilanadi.
- **Single shared `ResizeObserver`** — har item uchun alohida observer overhead. Bitta observer + `dataset.index` map. Browser support Chrome 64+, Firefox 69+, Safari 13.1+.
- **`react-window`** — Brian Vaughn library, lightweight, FixedSizeList/VariableSizeList/FixedSizeGrid. `style` prop majburiy, `React.memo` + `areEqual` per-item, `Row` komponentni module-level define qilish.
- **`@tanstack/react-virtual`** — modern alternative, hooks-based headless API, automatic measurement (`measureElement` ref), 2D support (rows + cols mustaqil virtualizers), framework-agnostic core.
- **Infinite scroll** — `useInfiniteQuery` (TanStack Query) + virtualization. `useEffect` last visible index list count'ga yaqin bo'lsa `fetchNextPage` trigger. Loader row va end row alohida states.
- **Sticky headers** — section header virtualization'da `position: sticky` ishlamasligi mumkin (transform context). Manual sticky simulation: scroll listener active section'ni hisoblaydi va alohida fixed element render qiladi.
- **Library tanlov:** `react-window` lightweight + simple API; `@tanstack/react-virtual` headless + 2D + automatic measurement. React Compiler era ikkalasi ham ishlaydi (Compiler virtual items render qilinadigan komponent'larga `useMemo`/`useCallback` ekvivalenti qo'llaydi).
- **Edge case'lar:** `key={index}` xato (state lost), `style` prop skip (overlap), container height yo'q (0 visible items), heavy work in scroll handler (lag), Strict Mode 2x render (production'da yo'q).

---

**Keyingi bo'lim:** [37-react-19-document-apis.md](37-react-19-document-apis.md) — React 19 Document & Resource APIs: Document Metadata (`<title>`/`<meta>`/`<link>` automatic `<head>` hoist, deduplication, react-helmet kerak emas), Stylesheet support (`<link rel="stylesheet" precedence>` Suspense bilan integration), Async Scripts (component-level `<script async>` automatic deduplication), Preloading APIs (`preload`/`preinit`/`prefetchDNS`/`preconnect`) chuqur tafsilotlar va programmatic ishlatish, SSR streaming integration, migration patterns.
