# Bo'lim 34: Profiling — DevTools va Production Telemetry

> Profiling — performance analizning **fundamental** vositasi. Bu fayl React DevTools Profiler chuqur (record, flame chart, ranked chart, commit details, "Why did this render?"), `<Profiler>` component programmatic API (onRender callback, production telemetry), production profiling build (`react-dom/profiling`), Web Vitals integration (LCP/INP/CLS), real-world workflow va Strict Mode'ning profile'ga ta'sirini o'rganadi. Performance optimization patterns'ni qo'llashdan oldin (cross-ref `33-optimization.md`) — **profile**. Re-render trigger'lar va Reconciler bailout (cross-ref `32-rendering-behavior.md`, `04-reconciliation.md`) — bu profile natijalarini tushuntiruvchi mexanizm. Render+Commit Phase timing va `useLayoutEffect` (cross-ref `02-rendering.md`, `17-uselayouteffect.md`) — profile'da `actualDuration`/`baseDuration` farqini izohlash uchun zarur.

---

## Mundarija

- [Profiling Concept va Importance](#profiling-concept-va-importance)
- [React DevTools Profiler — Installation va Interface](#react-devtools-profiler--installation-va-interface)
- [Recording Workflow](#recording-workflow)
- [Flame Chart Reading](#flame-chart-reading)
- [Ranked Chart View](#ranked-chart-view)
- [Commit Details va Filter](#commit-details-va-filter)
- ["Why did this render?" Panel](#why-did-this-render-panel)
- [`<Profiler>` Component API](#profiler-component-api)
- [onRender Callback Fields](#onrender-callback-fields)
- [Production Profiling — `react-dom/profiling`](#production-profiling--react-domprofiling)
- [Web Vitals Integration](#web-vitals-integration)
- [Real-World Workflow](#real-world-workflow)
- [Highlight Updates Option](#highlight-updates-option)
- [Strict Mode va Profile Impact](#strict-mode-va-profile-impact)
- [Performance Budget va Alerting](#performance-budget-va-alerting)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Profiling Concept va Importance

### Nazariya

**Profiling** — kod ishlash vaqti haqida **o'lchanadigan ma'lumot** to'plash. React'da profiling: komponent funksiyalari qancha vaqt oladi, qaysi komponentlar tez-tez re-render bo'ladi, qaysi commit'lar slow.

NIMA UCHUN profile zarur (cross-ref `33-optimization.md` Premature Optimization):

1. **Confirmation bias yo'q** — "Bu komponent slow bo'lishi kerak" intuition ko'pincha noto'g'ri.
2. **Hot path identifikatsiyasi** — Pareto principle, 5%/95%.
3. **Real-world impact** — sintetik benchmark va user metric'lar farq qiladi.
4. **Regression detection** — har optimization re-measure bilan tasdiqlanadi.
5. **Targeted optimization** — bottleneck topib, faqat shu joyga focus.

Profile turlari:

| Tur | Vosita | Maqsad |
|-----|--------|--------|
| **React-specific** | React DevTools Profiler | Re-render frequency, component duration |
| **JavaScript general** | Chrome DevTools Performance | JS execution flame chart, GC, microtasks |
| **Network** | Chrome Network tab | API calls, asset loading, waterfall |
| **Production user** | Web Vitals (LCP/INP/CLS) | Real user metrics |
| **Bundle** | Bundle analyzer (rollup-plugin-visualizer, vite-bundle-analyzer) | Code size, tree shaking |

Bu fayl asosan **React-specific profile** (DevTools Profiler + `<Profiler>` API + Web Vitals integration).

Profile workflow (cross-ref `33-optimization.md` Measure va Identify 7-step):

1. **Symptom** — user report yoki monitoring.
2. **Reproduce** — stepslarni aniqlash.
3. **Record** — DevTools Profiler.
4. **Analyze** — flame chart, "Why did this render?".
5. **Optimize** — targeted fix.
6. **Re-profile** — improvement confirmation.
7. **Production monitoring** — `<Profiler>` callback + Web Vitals.

NIMA UCHUN production monitoring kerak:

- Development build'da profile development behavior (Strict Mode 2x, DEV warning'lar overhead).
- Production build'da real user environment (low-end devices, network throttling, real data scale).
- Trend analysis — regression early detection.
- Alert thresholds — p95 commit duration > 50ms warning.

> **Eslatma:** Profile **continuous activity**, one-time emas. Performance budget va monitoring trend'ni tracking qilish — production app'larda essential.

<details>
<summary><strong>Under the Hood</strong></summary>

React DevTools internal API:

```javascript
// React har commit'da DevTools'ga ma'lumot uzatadi
window.__REACT_DEVTOOLS_GLOBAL_HOOK__.onCommitFiberRoot(rendererID, root);

// DevTools shu hook'dan Fiber tree'ni o'qib analiz qiladi
// - Per Fiber actualDuration
// - Per Fiber treeBaseDuration
// - Lanes priority info
// - Effect flags
```

Profiler ma'lumotlari Fiber struct'idan:

```typescript
interface Fiber {
  // ...
  actualDuration: number;     // Bu commit'da Fiber + descendants render time
  actualStartTime: number;    // Render boshlangan vaqt
  selfBaseDuration: number;   // Faqat shu Fiber render vaqti (descendants emas)
  treeBaseDuration: number;   // Subtree render vaqti (memoization'siz)
}
```

Production build'da bu maydon'lar **default'da yo'q** (`production` mode'da memory tejaladi). `react-dom/profiling` build esa maydon'larni saqlaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Sodda profiling setup:

```tsx
import { Profiler } from 'react';
import type { ReactElement, ProfilerOnRenderCallback } from 'react';

const onRender: ProfilerOnRenderCallback = (
  id,
  phase,
  actualDuration,
  baseDuration,
  startTime,
  commitTime
) => {
  console.log(`[${id}] ${phase}: ${actualDuration.toFixed(2)}ms`);
};

function App(): ReactElement {
  return (
    <Profiler id="App" onRender={onRender}>
      <Header />
      <MainContent />
      <Footer />
    </Profiler>
  );
}
// Output har commit'da:
// [App] mount: 5.23ms
// [App] update: 1.45ms
// [App] update: 0.89ms (mostly bailouts)
```

</details>

---

## React DevTools Profiler — Installation va Interface

### Nazariya

**React DevTools** — Facebook (Meta) tomonidan rasmiy maintained browser extension. Components tab (tree, props, state, hooks) va Profiler tab (record, analyze).

### Install

| Browser | Source |
|---------|--------|
| Chrome | Chrome Web Store — "React Developer Tools" |
| Firefox | Firefox Add-ons — "React Developer Tools" |
| Edge | Edge Add-ons — "React Developer Tools" |
| Safari | Limited (no official extension; safari-react-devtools npm) |
| Standalone | `react-devtools` npm package (Electron app) |

Standalone usage (React Native, mobile, embedded):

```bash
npm install -g react-devtools
react-devtools
# DevTools window ochiladi, port 8097
```

App'da connection:

```html
<!-- index.html -->
<script src="http://localhost:8097"></script>
```

### Interface

DevTools panel (F12) — ikkita React tab:

```
┌─────────────────────────────────────────────────┐
│ Elements │ Console │ Components │ Profiler │ ... │
└─────────────────────────────────────────────────┘
```

**Components tab:**

- Tree view — Fiber tree (komponent hierarchy'si).
- Selected komponent: props, state, hooks, owner, source.
- Search bar — komponent name bo'yicha filter.
- Settings — display options.

**Profiler tab:**

- Record button (circle).
- Settings (gear icon).
- Flame chart / Ranked chart view selector.
- Commit slider (record bo'lgan commits navigation).
- Component details panel.

### Settings Yoqish (Tavsiya etiladi)

Profile to'liq foydali bo'lishi uchun bitta setting majburiy, ikkinchisi alohida visual debug feature:

```
F12 → DevTools panel → Settings (gear icon)

Profiler tab settings:
  ☑ "Record why each component rendered while profiling"
     - Profile recording'ga "Why did this render?" panel'ni qo'shadi
     - Overhead: har commit'da extra fiber metadata (yuqorida UTH section'da)

Components tab settings (alohida feature):
  ☑ "Highlight updates when components render"
     - Real-time visual feedback (rectangle border)
     - Profile recording'dan mustaqil — har doim ishlaydi (DevTools yoniq bo'lsa)
```

**`Record why each component rendered`** — overhead bor (commit'da DevTools heap o'sadi), faqat profile session vaqtida yoqish. Production'da DevTools odatda foydalanuvchilarda yo'q.

> **Eslatma:** Highlight Updates Profile recording'dan ALOHIDA feature — Components tab settings ostida. Profile uchun majburiy emas, lekin "qaysi komponent qachon re-render bo'lyapti" deb tezkor visual scan uchun foydali.

### Components vs Profiler Tab Farqi

| Tab | Maqsad | When to use |
|-----|--------|-------------|
| Components | State/props inspection, manual testing | Debug logic, state tracking |
| Profiler | Performance analysis, re-render tracking | Optimization workflow |

Profile uchun — Profiler tab. Components tab faqat manual debug.

> **Eslatma:** DevTools versiyasi React versiyasi bilan **mos** bo'lishi shart. R19 yangi feature'lar uchun (use-promise marker, Compiler-generated hooks display) yangi DevTools kerak. Auto-update Chrome Web Store orqali.

<details>
<summary><strong>Under the Hood</strong></summary>

DevTools-React communication:

```
Browser DevTools panel
   │
   │ chrome.runtime API
   │
   ▼
Background script
   │
   │ Page-injected script
   │
   ▼
__REACT_DEVTOOLS_GLOBAL_HOOK__
   │
   │ Hook'da React register
   │
   ▼
React (development build)
   - onCommitFiberRoot, onCommitFiberUnmount
   - DEV warning'lar via dispatcher
```

`__REACT_DEVTOOLS_GLOBAL_HOOK__` window'da global. React init paytida shu hook'ni topib, DevTools'ga subscribe.

Production build'da React DevTools register qilmaydi maxsus development info (props names, hooks display, "Why did this render"), faqat minimal info.

`react-devtools-inline` npm — embedded usage uchun:

```javascript
import { initialize, activate } from 'react-devtools-inline/frontend';

// Embedded DevTools (e.g., StackBlitz, CodeSandbox)
const root = initialize(window);
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Standalone DevTools setup:

```javascript
// package.json
{
  "scripts": {
    "devtools": "react-devtools"
  },
  "devDependencies": {
    "react-devtools": "^5.0.0"
  }
}
```

```bash
# Terminal 1: standalone DevTools
npm run devtools
# DevTools window opens, listens on port 8097

# Terminal 2: app
npm run dev
```

```html
<!-- index.html — connect script -->
<head>
  <script src="http://localhost:8097"></script>
</head>
```

Production-only check:

```typescript
// vite.config.ts
import { defineConfig } from 'vite';

export default defineConfig({
  // Development'da DevTools script yuklash, production'da yo'q
  define: {
    __DEVTOOLS_ENABLED__: process.env.NODE_ENV === 'development',
  },
});
```

```html
<head>
  <script>
    if (typeof __DEVTOOLS_ENABLED__ !== 'undefined' && __DEVTOOLS_ENABLED__) {
      const script = document.createElement('script');
      script.src = 'http://localhost:8097';
      document.head.appendChild(script);
    }
  </script>
</head>
```

</details>

---

## Recording Workflow

### Nazariya

Recording — Profiler tab'da aniq performance interval'ni o'lchash. Workflow 3 qadam:

### Qadam 1: Reproduce Stepslarni Aniqlash

Profile'dan oldin **aniq scenario** kerak:

```
✅ Yaxshi:
   "User typing in search box laggy"
   Reproduce: 1) Open dashboard 2) Type "test" 3) Notice lag

✅ Yaxshi:
   "Modal open animation drops frames"
   Reproduce: 1) Click "Open Modal" button 2) Watch animation

❌ Yomon:
   "App is slow"
   - Aniqroq stepslar kerak
```

### Qadam 2: Record

```
1. F12 → Profiler tab
2. Click record button (circle)
   - Button qizil → recording faol
3. Reproduce stepslar perform
   - Type, click, scroll
4. Click record button qaytadan (square)
   - Recording stops
   - Flame chart ko'rinadi
```

### Qadam 3: Initial View

Recording natija:

- **Commit slider** — har commit (har render → DOM commit) alohida.
- **Flame chart** — joriy commit'dagi komponent tree.
- **Color coding**:
  - Yashil — fast render
  - Sariq — medium
  - Qizil — slow render

Slider'ni o'ngga/chap'ga yurgizib, har commit'ni ko'rish.

### Recording Tips

1. **Production-like data** — minimal data bilan profile yetarli emas. 1000+ items list, real user data hajmida.
2. **Realistic interaction** — fast typing, scroll, click sequences.
3. **Network throttling** — Chrome DevTools Network tab → Throttling → "Slow 3G" — real user condition'da test.
4. **CPU throttling** — Chrome DevTools Performance tab → CPU → "4x slowdown" — low-end device simulation.
5. **Strict Mode disable for accurate measurement** — development'da Strict Mode 2x render. Production hisoblash uchun olib tashlash (lekin bug surface bo'lishi mumkin, ehtiyot).

### Recording Limits

- **Memory cost** — har commit DevTools heap'ga saqlanadi. Uzun recording'da (yuzlab–minglab commits) heap sezilarli darajada o'sadi; DevTools chegaraga yetganda warning ko'rsatadi va eski commit'larni truncate qiladi.
- **Time limit** — 30+ second recording'lar DevTools'da slowness keltirib chiqaradi.
- **Tab switch** — recording paytida tab switch qilinmaydi (data loss).

Tavsiya: 5-10 second recording'lar, fokuslangan reproduce.

<details>
<summary><strong>Under the Hood</strong></summary>

DevTools recording — har `onCommitFiberRoot` chaqirig'ida data captures:

```javascript
// React internal (development)
function commitRoot(root) {
  // ...
  const commitTime = now();
  
  if (__DEV__) {
    // DevTools hook chaqirig'i
    window.__REACT_DEVTOOLS_GLOBAL_HOOK__.onCommitFiberRoot(
      rendererID,
      root,
      priorityLevel,
      isStrictMode
    );
  }
}

// DevTools side
function onCommitFiberRoot(rendererID, root) {
  if (recording) {
    const snapshot = serializeFiberTree(root);
    commits.push({
      timestamp: now(),
      duration: root.actualDuration,
      tree: snapshot,
      changedFibers: collectChangedFibers(root),
    });
  }
}
```

Memory growth (sifat-darajada — aniq qiymat fiber tree o'lchami, "Why did this render" settings va React versiyasiga bog'liq):

- Har commit'da Fiber tree snapshot, per-fiber render reason metadata va hooks/effects state serialize qilinadi.
- Kichik tree (o'nlab fiber) — har commit kilobyte miqyosida; katta tree (yuzlab–minglab fiber, "Record why each component rendered" yoqilgan) — har commit yuzlab kilobyte.
- Recording davomida bu data accumulate bo'ladi va DevTools heap'iga saqlanadi; uzun sessiyalarda DevTools tab'ning o'zi sekinlashishi mumkin.

Recording stop bo'lganda — data parse qilinadi, flame chart render bo'ladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Recording-ready test scenario:

```tsx
// Profile-friendly app structure
import { useState } from 'react';
import type { ReactElement } from 'react';

interface User {
  id: string;
  name: string;
  email: string;
}

// Production-like data
function generateUsers(count: number): User[] {
  return Array.from({ length: count }, (_, i) => ({
    id: `user-${i}`,
    name: `User ${i}`,
    email: `user${i}@example.com`,
  }));
}

function ProfileTestPage(): ReactElement {
  const [users] = useState(() => generateUsers(1000)); // 1000 users
  const [search, setSearch] = useState('');
  
  // Reproduce: type fast in input, observe profile
  return (
    <div>
      <input
        value={search}
        onChange={(e) => setSearch(e.target.value)}
        placeholder="Type to filter..."
      />
      <UserList users={users} filter={search} />
    </div>
  );
}

function UserList({ users, filter }: { users: User[]; filter: string }): ReactElement {
  const filtered = users.filter((u) => u.name.includes(filter));
  return (
    <ul>
      {filtered.map((u) => (
        <li key={u.id}>{u.name} — {u.email}</li>
      ))}
    </ul>
  );
}

// Recording workflow:
// 1. Open browser DevTools → Profiler tab
// 2. Click record (circle)
// 3. In app: type "test" character by character
// 4. Click stop (square)
// 5. Analyze flame chart
```

CPU throttling for realistic profile:

```typescript
// Chrome DevTools → Performance tab → CPU dropdown:
// - "No throttling" — local development
// - "4x slowdown" — average mid-range device
// - "6x slowdown" — low-end device

// Network tab → Throttling:
// - "No throttling"
// - "Fast 3G" — 1.5 Mbps
// - "Slow 3G" — 400 Kbps
// - Custom — defined latency + bandwidth
```

</details>

---

## Flame Chart Reading

### Nazariya

**Flame chart** — har commit'dagi komponent tree'ni vizual ko'rsatish. X-axis: vaqt, Y-axis: komponent depth. Har bar — bitta komponent.

### Reading Rules

```
┌────────────────────────────────────────────┐
│ App (5.2ms)                                │ ← Root, total commit duration
├────────────────────────────────────────────┤
│ Header (0.5ms)│ MainContent (4.5ms)        │ ← Children, parallel display
├───────────────┴──────────┬─────────────────┤
│                          │ UserList (4ms)  │
│                          ├─────────────────┤
│                          │ Item│Item│Item│ │ ← Leaf'lar
└──────────────────────────┴─────────────────┘
```

- **Width** — render duration (wider = slower).
- **Color** — bu commit'ning eng sekin fiber'iga **nisbatan** rang scale:
  - Gray — bailout (render qilmagan)
  - Cool tones (teal/blue) — joriy commit'da nisbatan tez fiber'lar
  - Warm tones (yellow/orange) — joriy commit'da nisbatan sekin fiber'lar
  - Eng "issiq" rang shu commit'ning eng sekin fiber'iga to'g'ri keladi; absolute ms threshold yo'q
- **Tooltip** — hover bilan aniq duration (ms) ko'rsatiladi.

> **Eslatma:** Rang **commit ichidagi relative position**'ga bog'liq. Agar bir commit'da hamma fiber 50ms+ bo'lsa, "tez" rang ham 30ms fiber'ga tushishi mumkin. Frame budget tahlili uchun absolute ms threshold (16ms = 60fps frame) tooltip orqali tekshiriladi, rang scale orqali emas.

### Klik Behavior

- **Bar click** — komponent details panel'ni o'ngda ochadi.
- **Bar double-click** — Components tab'ga shu komponent bilan switch.
- **Hover** — tooltip with full info.

### Top-Down Reading

Flame chart top'dan boshlanadi (root) va pastga. Reading order:

1. **Root** — total commit duration. Agar > 16ms — frame drop.
2. **Wide children** — bottleneck candidates.
3. **Deep narrow** — komponent hierarchy tracking.
4. **Gray bars** — bailout bo'lgan, performance positive.

Misol — slow commit:

```
App (50ms) ← Slow commit
├─ Header (0.5ms) [gray] — bailout, fast
└─ Dashboard (49ms) ← Bottleneck
   ├─ Stats (1ms)
   ├─ Chart (5ms)
   └─ Table (43ms) ← Real bottleneck
      ├─ Row 1 (0.5ms)
      ├─ Row 2 (0.5ms)
      ... (1000 rows × 0.5ms = 500ms — but Reconciler interleaved)
```

Bottleneck — `Table` (1000+ rows). Optimization: virtualization (cross-ref 36) yoki memo.

### Width Interpretation

```
Bar width = duration / total_commit_duration

50ms total commit:
   Header bar = 0.5/50 = 1% width
   Dashboard bar = 49/50 = 98% width
   
Visually: Header thin, Dashboard wide
```

### Self vs Tree Duration

DevTools 2 ta metric:

- **Self** — komponent o'zining render time (descendants emas).
- **Tree** — komponent + descendants render time.

```
Dashboard:
   Self = 1ms (Dashboard funksiyasi)
   Tree = 49ms (Dashboard + Stats + Chart + Table)
```

Bottleneck — **wide self time** (komponent o'zi slow). Wide tree, narrow self — child bottleneck.

<details>
<summary><strong>Under the Hood</strong></summary>

Flame chart algoritm:

```javascript
// DevTools internal — render flame chart
function renderFlameChart(commit, container) {
  const totalDuration = commit.duration;
  const containerWidth = container.clientWidth;
  // Commit ichidagi eng sekin fiber — rang scale uchun anchor
  const maxFiberDuration = findMaxActualDuration(commit.root);
  
  function renderFiber(fiber, x, y, depth) {
    const width = (fiber.actualDuration / totalDuration) * containerWidth;
    const height = 20; // Fixed bar height
    
    drawRect(x, y + depth * height, width, height, getColor(fiber.actualDuration, maxFiberDuration));
    drawText(x + 4, y + depth * height + 12, `${fiber.componentName} (${fiber.actualDuration.toFixed(2)}ms)`);
    
    // Recurse children
    let childX = x;
    for (const child of fiber.children) {
      renderFiber(child, childX, y, depth + 1);
      childX += (child.actualDuration / totalDuration) * containerWidth;
    }
  }
  
  renderFiber(commit.root, 0, 0, 0);
}

function getColor(duration, maxDuration) {
  if (duration === 0) return '#aaa'; // Gray (bailout)
  // Joriy commit'ga nisbatan intensity (0..1)
  const intensity = maxDuration > 0 ? duration / maxDuration : 0;
  // HSL hue mapping: cool (teal ~180°) → hot (red ~0°)
  const hue = (1 - intensity) * 180;
  return `hsl(${hue}, 70%, 50%)`;
}
```

Bailout fibers — `actualDuration === 0`, gray render.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Flame chart interpretation example:

```tsx
import { useState, memo } from 'react';
import type { ReactElement } from 'react';

const FastChild = memo(function FastChild({ value }: { value: number }): ReactElement {
  return <span>Fast: {value}</span>;
});

function SlowChild({ items }: { items: number[] }): ReactElement {
  // Imitatsiya: O(n^2) operation
  let sum = 0;
  for (let i = 0; i < items.length; i++) {
    for (let j = 0; j < items.length; j++) {
      sum += items[i] * items[j];
    }
  }
  return <span>Slow: {sum}</span>;
}

function App(): ReactElement {
  const [count, setCount] = useState(0);
  const [items] = useState(() => Array.from({ length: 1000 }, (_, i) => i));
  
  return (
    <>
      <button onClick={() => setCount((c) => c + 1)}>Click: {count}</button>
      <FastChild value={42} />
      <SlowChild items={items} />
    </>
  );
}
// Profile flame chart:
// App (~10ms)
//   ├─ button (0.5ms) [updated text]
//   ├─ FastChild (0ms) [gray — memo bailout, value=42 unchanged]
//   └─ SlowChild (~9ms) ← Yellow/red, bottleneck
//
// Optimization: SlowChild ni memo yoki useMemo ichida sum'ni cache
```

</details>

---

## Ranked Chart View

### Nazariya

**Ranked chart** — flame chart'ning alternative view. Komponent'lar **render duration bo'yicha tartiblangan**, eng sekin tepada.

DevTools Profiler tab'da chart turini selector orqali tanlash:

```
[Flame Chart] [Ranked Chart] ← Selector
```

### Format

```
┌────────────────────────────────────────────┐
│ SlowComponent (43.5ms) ━━━━━━━━━━━━━━━━━━ │ ← Eng sekin
├────────────────────────────────────────────┤
│ Chart (5.2ms) ━━━━━━━━                      │
├────────────────────────────────────────────┤
│ Header (0.8ms) ━                            │
├────────────────────────────────────────────┤
│ Footer (0.3ms) ━                            │
└────────────────────────────────────────────┘
```

- **Y-axis** — komponent ranking.
- **X-axis** — duration (proportional bar width).
- **Order** — slowest first (descending).

### NIMA UCHUN Ranked Chart

Flame chart — **structural** view (parent-child).
Ranked chart — **performance** view (slowest first).

Use cases:

- **Bottleneck identification** — first bar = main culprit.
- **Optimization priority** — top 3-5 components first.
- **Quick scan** — flame chart deep tree complex, ranked simple.

### Limitations

- **No hierarchy** — komponent context yo'qoladi.
- **Same component duplicated** — har Render ko'rinishi alohida.
- **Less context** — qaysi parent re-render trigger qilgani ko'rinmaydi.

Optimization workflow:

```
1. Ranked chart — top component identify
2. Flame chart switch — context to'liq tushunish
3. Click component — details panel
4. "Why did this render?" — sabab
```

<details>
<summary><strong>Under the Hood</strong></summary>

Ranked chart algorithm:

```javascript
function renderRankedChart(commit, container) {
  const fibers = collectAllFibers(commit.root);
  
  // Sort by actualDuration descending
  fibers.sort((a, b) => b.actualDuration - a.actualDuration);
  
  // Filter bailouts
  const rendered = fibers.filter((f) => f.actualDuration > 0);
  
  const maxDuration = rendered[0]?.actualDuration ?? 1;
  
  rendered.forEach((fiber, i) => {
    const width = (fiber.actualDuration / maxDuration) * container.clientWidth;
    drawRect(0, i * 20, width, 18, getColor(fiber.actualDuration));
    drawText(4, i * 20 + 12, `${fiber.componentName} (${fiber.actualDuration.toFixed(2)}ms)`);
  });
}
```

Bailout fibers (actualDuration === 0) ranked chart'ga kirmaydi — performance fokus.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Ranked chart use case scenario:

```tsx
import { useState, memo } from 'react';
import type { ReactElement } from 'react';

// Tree:
// App
//   ├─ Header
//   ├─ MainContent
//   │   ├─ Sidebar
//   │   └─ ContentArea
//   │       ├─ Editor (slow!)
//   │       └─ Preview
//   └─ Footer

function SlowEditor(): ReactElement {
  // Imitatsiya: heavy syntax highlighting
  for (let i = 0; i < 1000000; i++) { /* ... */ }
  return <textarea />;
}

const Header = memo(() => <header>Header</header>);
const Sidebar = memo(() => <aside>Sidebar</aside>);
const Preview = memo(() => <div>Preview</div>);
const Footer = memo(() => <footer>Footer</footer>);

function App(): ReactElement {
  const [count, setCount] = useState(0);
  
  return (
    <>
      <button onClick={() => setCount((c) => c + 1)}>{count}</button>
      <Header />
      <Sidebar />
      <SlowEditor />
      <Preview />
      <Footer />
    </>
  );
}
// Ranked chart on click:
// 1. SlowEditor (~50ms) ← Ranked first, bottleneck
// 2. App (~0.5ms)
// 3. button (~0.1ms)
// (Header, Sidebar, Preview, Footer — bailout, ranked'da yo'q)
//
// Optimization priority: SlowEditor first
```

</details>

---

## Commit Details va Filter

### Nazariya

**Commit details panel** — flame chart'da komponent tanlash → o'ngda detail. Ma'lumot:

```
┌─────────────────────────────────────┐
│ Component: UserList                 │
│ Phase: update                       │
│ Render duration: 12.3ms             │
│ Self time: 2.1ms                    │
│ Tree time: 12.3ms                   │
│                                     │
│ Why did this render?                │
│ • Hooks changed (state)             │
│ • Props changed: filter             │
│                                     │
│ Props:                              │
│ ▸ users: [...]                      │
│ ▸ filter: "test"                    │
│                                     │
│ State:                              │
│ ▸ sortBy: "name"                    │
│                                     │
│ Hooks:                              │
│ 1. State (sortBy: "name")           │
│ 2. Memo [filtered array]            │
│ 3. Callback [handleSort]            │
└─────────────────────────────────────┘
```

### Sections

1. **Component info** — name, phase (mount/update), source location.
2. **Duration** — render time, self time, tree time.
3. **Why did this render?** — re-render sabablari (settings yoqilgan bo'lsa).
4. **Props** — joriy props, eski props bilan diff.
5. **State** — joriy state, eski bilan diff.
6. **Hooks** — barcha hook'lar tartibda, qiymatlari bilan.

### Commit Filter

DevTools commit slider'da filter:

```
[All commits] [Slow only] [State only] [...]
```

Filter options:

- **All commits** — har bir commit (default).
- **Slow only** — duration > X ms (configurable).
- **State changes only** — useState/useReducer trigger qilgan commit'lar.
- **Component-specific** — faqat ma'lum komponent re-rendered commit'lar.

NIMA UCHUN filter:

- 1000+ commits navigate qilish qiyin.
- Slow commits focus — eng katta impact.
- Specific komponent debug — faqat shu komponent o'zgarishlari.

### Diff View

Props/state changed indicators:

```
Props:
   users: [...]  ← Same (no diff)
   filter: "tes" → "test"  ← Changed (highlighted yellow)
```

DevTools change'ni inline ko'rsatadi (yangi vs eski).

### Source Location

Komponent source file/line:

```
Source: /src/components/UserList.tsx:42:8
```

Click — IDE'da open (DevTools settings'da configure):

```
Settings → "Open in Editor"
   Pattern: vscode://file/{path}:{line}:{column}
```

<details>
<summary><strong>Under the Hood</strong></summary>

Commit details — Fiber inspection:

```javascript
function inspectFiber(fiber) {
  return {
    componentName: fiber.elementType?.displayName || fiber.elementType?.name,
    phase: fiber.alternate === null ? 'mount' : 'update',
    actualDuration: fiber.actualDuration,
    selfBaseDuration: fiber.selfBaseDuration,
    treeBaseDuration: fiber.treeBaseDuration,
    
    props: serializeProps(fiber.memoizedProps),
    state: serializeState(fiber.memoizedState),
    hooks: serializeHooks(fiber.memoizedState),
    
    source: extractSourceLocation(fiber),
  };
}
```

Hooks serialization — DevTools Hook chain'ni traverse qilib, har hook turini display qiladi:

```javascript
function serializeHooks(memoizedState) {
  const hooks = [];
  let current = memoizedState;
  let index = 0;
  
  while (current !== null) {
    hooks.push({
      index,
      type: detectHookType(current),
      value: serializeValue(current),
    });
    current = current.next;
    index++;
  }
  
  return hooks;
}
```

`detectHookType` — heuristic: `current.queue` bor → useState/useReducer, `current.deps` bor → useMemo/useCallback, va h.k.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

DevTools-friendly component (display names):

```tsx
import { memo, forwardRef } from 'react';
import type { ReactElement } from 'react';

// Anonymous memo — DevTools shows "Memo"
const Anonymous = memo(({ value }: { value: string }) => <span>{value}</span>);
// DevTools tree: "Memo" (less helpful)

// Named — better
const NamedComponent = memo(function NamedComponent({ value }: { value: string }) {
  return <span>{value}</span>;
});
// DevTools tree: "Memo(NamedComponent)" — clear

// Or explicit displayName
const ExplicitNamed = memo(({ value }: { value: string }) => <span>{value}</span>);
ExplicitNamed.displayName = 'ExplicitNamed';
// DevTools tree: "Memo(ExplicitNamed)"

// forwardRef
const FancyButton = forwardRef<HTMLButtonElement, { label: string }>(function FancyButton(
  { label },
  ref
): ReactElement {
  return <button ref={ref}>{label}</button>;
});
// DevTools: "ForwardRef(FancyButton)"
```

</details>

---

## "Why did this render?" Panel

### Nazariya

**"Why did this render?"** — Profiler'ning eng foydali feature'i. Re-render sabablarini explicit ko'rsatadi.

### Reasons

| Reason | Tasvirlash | Optimization Strategy |
|--------|-----------|----------------------|
| **The first time the component rendered** | Initial mount | Normal — komponent yangi mount |
| **Hooks changed** | useState/useReducer state change | Normal — state update'ga response |
| **Props changed: (props list)** | Parent yangi props berdi | useMemo/useCallback parent'da |
| **Parent component rendered** | Top-down propagation, props bir xil | React.memo |
| **Context changed** | useContext value change | Context split / selector |

### Multiple Reasons

Bir commit'da bir nechta sabab:

```
Why did this render?
• Hooks changed (state: count)
• Props changed: items, filter
• Context changed
```

Bu komponent re-render bo'lishining **barcha trigger'lari**.

### Props Diff Detail

`Props changed` reason ostida — qaysi props o'zgarganini ko'rsatadi:

```
Props changed:
   • items (new reference, shallow equal: false)
   • filter ("tes" → "test")
```

Optimization workflow:

1. `items` — agar real o'zgarish bo'lsa OK; agar yangi reference but shape bir xil bo'lsa — `useMemo` parent'da.
2. `filter` — primitive value o'zgargan, real change.

### Hooks Changed Detail

```
Hooks changed:
   • State 1: 0 → 1 (count)
   • Memo 2: [array of 100 items] → [array of 100 items] (new reference)
```

Memo dep yangi reference — `useMemo` recomputed. Stable reference ta'minlash kerak (cross-ref `33-optimization.md`).

### Parent Component Rendered

```
Parent component rendered (no props change)
```

Top-down propagation — komponent props o'zgarmagan, lekin parent re-rendered. `React.memo` bilan to'xtatish mumkin.

### Settings Yoqish

```
Profiler tab → Settings (gear) → ☑ "Record why each component rendered while profiling"
```

**MUHIM**: settings yoqilganda **performance overhead** bor (DevTools har commit'da extra metadata saqlaydi). Faqat profile vaqtida yoqish, production'da off.

<details>
<summary><strong>Under the Hood</strong></summary>

"Why did this render?" — Fiber's previous vs current props/state diff:

```javascript
function detectRenderReasons(fiber) {
  const reasons = [];
  
  if (fiber.alternate === null) {
    reasons.push('First render (mount)');
    return reasons;
  }
  
  // Hooks changed
  const hookChanges = detectHookChanges(fiber);
  if (hookChanges.length > 0) {
    reasons.push(`Hooks changed: ${hookChanges.join(', ')}`);
  }
  
  // Props changed
  const propChanges = detectPropChanges(fiber.alternate.memoizedProps, fiber.memoizedProps);
  if (propChanges.length > 0) {
    reasons.push(`Props changed: ${propChanges.join(', ')}`);
  }
  
  // Context changed
  if (hasContextChanged(fiber)) {
    reasons.push('Context changed');
  }
  
  // No props/hooks/context — parent rendered
  if (reasons.length === 0) {
    reasons.push('Parent component rendered');
  }
  
  return reasons;
}

function detectPropChanges(prevProps, nextProps) {
  const changed = [];
  for (const key of Object.keys(nextProps)) {
    if (!Object.is(prevProps[key], nextProps[key])) {
      changed.push(key);
    }
  }
  return changed;
}
```

DevTools shu data'ni capture qiladi har commit'da (overhead — settings yoqilganda).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Debug session example:

```tsx
import { useState, useMemo, memo } from 'react';
import type { ReactElement } from 'react';

interface Item {
  id: string;
  name: string;
}

const ItemRow = memo(function ItemRow({ item }: { item: Item }): ReactElement {
  return <li>{item.name}</li>;
});

function ItemList({ items, filter }: { items: Item[]; filter: string }): ReactElement {
  const filtered = items.filter((i) => i.name.includes(filter));
  return (
    <ul>
      {filtered.map((i) => (
        <ItemRow key={i.id} item={i} />
      ))}
    </ul>
  );
}

function App(): ReactElement {
  const [items, setItems] = useState<Item[]>([
    { id: '1', name: 'Apple' },
    { id: '2', name: 'Banana' },
  ]);
  const [filter, setFilter] = useState('');
  const [counter, setCounter] = useState(0);
  
  return (
    <>
      <input value={filter} onChange={(e) => setFilter(e.target.value)} />
      <button onClick={() => setCounter((c) => c + 1)}>Counter: {counter}</button>
      <ItemList items={items} filter={filter} />
    </>
  );
}

// Profile session 1 — counter button bosilganda (filter o'zgarmagan):
// 1. Click counter button → App re-renders (counter state changed)
// 2. Stop record
// 3. Inspect ItemList:
//    Why did this render?
//    • Parent component rendered (memo'siz; props.items, props.filter o'zgarmagan, lekin parent App re-rendered)
// 4. Inspect ItemRow #1:
//    Bailout — ItemRow memo, props.item === oldProps.item (items reference stable).
//    Why did this render panel'da ItemRow ko'rinmaydi (render bo'lmagan).

// Profile session 2 — input'ga "App" yozilganda:
// 1. App:
//    Why did this render?
//    • Hooks changed (state: filter — App'ning useState)
// 2. ItemList:
//    Why did this render?
//    • Props changed: filter (App'dan kelgan yangi prop)
//    (Hooks changed YO'Q — ItemList'ning o'z useState'i yo'q)
// 3. ItemRow #1 (Apple): bailout — item reference bir xil (items array stable), memo bailout
// 4. ItemRow #2 (Banana): unmount — filter qilingan natijadan tushib qoldi
// 5. Reconciler key match qiladi (id-based stable identity)
```

</details>

---

## `<Profiler>` Component API

### Nazariya

**`<Profiler>`** — programmatic profiling API. React rasmiy export, DevTools'siz production telemetry uchun.

```tsx
import { Profiler } from 'react';

<Profiler id="MyComponent" onRender={onRenderCallback}>
  <MyComponent />
</Profiler>
```

### Props

| Prop | Type | Description |
|------|------|-------------|
| `id` | string | Profiler identifier (har Profiler unique) |
| `onRender` | function | Har commit'da chaqiriladi (callback) |

### Nesting

`<Profiler>` nested bo'lishi mumkin:

```tsx
<Profiler id="App" onRender={onRender}>
  <Profiler id="Header" onRender={onRender}>
    <Header />
  </Profiler>
  <Profiler id="Main" onRender={onRender}>
    <MainContent />
  </Profiler>
</Profiler>
```

Har commit'da har Profiler `onRender` chaqiriladi (granular monitoring).

### Performance Impact

Development build:
- `<Profiler>` overhead — fiber commit hook + `onRender` chaqirig'i (commit'dagi boshqa DEV warning'lar oldida sezilarli emas).
- DevTools panel'da `<Profiler>` Fiber ko'rinadi.

Production build:
- `<Profiler>` Fiber yaratiladi, lekin `commitProfiler` instrumentation kod yo'li bundle'ga kirmaydi — `onRender` chaqirilmaydi (no-op timing).
- Profile data measure qilish uchun — `react-dom/profiling` build kerak (Profiler instrumentation production'da ham yoqilgan variant).

### Use Cases

1. **Production telemetry** — real user metrics monitoring (Datadog, Sentry, custom).
2. **Performance budget enforcement** — alert agar slow render.
3. **A/B testing** — feature flag bilan optimization compare.
4. **Regression detection** — trend analysis.

### NIMA QILMAYDI

`<Profiler>` ham:

- **Optimization qilmaydi** — faqat measure.
- **Re-render trigger qilmaydi** — pure pass-through.
- **State o'zgartirmaydi** — read-only API.

`<Profiler>` — **measurement tool**, mo'ljallanadi production'da minimal overhead bilan.

> **Eslatma:** `<Profiler>` R16.5 (2018-09) DevTools integration uchun joriy etilgan, R16.9 (2019-08) `react` package'dan rasmiy export sifatida stable. R18+'da `nested-update` phase qiymati qo'shildi (`useLayoutEffect`/`useEffect` ichidagi `setState` natijasidagi commit'lar).

<details>
<summary><strong>Under the Hood</strong></summary>

`<Profiler>` JSX'da maxsus type — `react` package'dan `Profiler` import qilinganda `$$typeof: REACT_PROFILER_TYPE` Symbol bilan element yaratiladi. Reconciler shu Symbol'ni tanib `Profiler` Fiber tag (cross-ref `03-fiber-architecture.md`) bilan Fiber yaratadi. Bu **funksional komponent emas** — JS function body chaqirilmaydi; barcha xatti-harakat React internal Fiber lifecycle hook'larida.

Element yaratish:

```javascript
// react/src/ReactProfiler.js (oddiylashtirilgan)
export const Profiler = Symbol.for('react.profiler');

// jsx() Profiler'ni tanib element yaratadi:
// {
//   $$typeof: REACT_ELEMENT_TYPE,
//   type: Profiler,  // Symbol
//   props: { id, onRender, children }
// }
```

Production build (`react-dom.production.min.js`):

```javascript
// commitWork — Profiler Fiber ko'radi, lekin instrumentation kod yo'li bundle'ga kirmaydi
function commitWork(fiber) {
  // ... boshqa fiber tag'lar
  if (fiber.tag === Profiler) {
    // Production: faqat children commit, onRender chaqirilmaydi
    // (treeBaseDuration/actualDuration field'lari ham hisoblanmaydi)
  }
  // ... children commit qilinadi (Fiber subtree normal ishlaydi)
}
```

`react-dom/profiling` build (`react-dom.profiling.min.js`):

```javascript
// commitWork — Profiler instrumentation yoqilgan
function commitWork(fiber) {
  if (fiber.tag === Profiler) {
    const { onRender, id } = fiber.memoizedProps;
    
    if (typeof onRender === 'function') {
      const phase = fiber.alternate === null
        ? 'mount'
        : (isNestedUpdate ? 'nested-update' : 'update');
      
      onRender(
        id,
        phase,
        fiber.actualDuration,
        fiber.treeBaseDuration,
        fiber.actualStartTime,
        currentCommitTime
      );
    }
  }
}
```

`onRender` chaqiriladi har commit'da, agar `<Profiler>` subtree'da render bo'lgan bo'lsa (`actualDuration > 0`). Pure bailout commit'larda (subtree butunlay skip'da) `onRender` chaqirilmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Multiple Profilers:

```tsx
import { Profiler } from 'react';
import type { ReactElement, ProfilerOnRenderCallback } from 'react';

const onRender: ProfilerOnRenderCallback = (id, phase, actualDuration) => {
  console.log(`[${id}] ${phase}: ${actualDuration.toFixed(2)}ms`);
};

function App(): ReactElement {
  return (
    <Profiler id="App" onRender={onRender}>
      <Profiler id="Navigation" onRender={onRender}>
        <Navigation />
      </Profiler>
      
      <main>
        <Profiler id="ProductList" onRender={onRender}>
          <ProductList />
        </Profiler>
        
        <Profiler id="Sidebar" onRender={onRender}>
          <Sidebar />
        </Profiler>
      </main>
      
      <Profiler id="Footer" onRender={onRender}>
        <Footer />
      </Profiler>
    </Profiler>
  );
}

// Console output har commit'da:
// [Navigation] mount: 0.5ms
// [ProductList] mount: 12.3ms
// [Sidebar] mount: 2.1ms
// [Footer] mount: 0.3ms
// [App] mount: 15.8ms ← root, includes nested
```

Conditional profiling:

```tsx
import { Profiler, type ReactNode } from 'react';

const PROFILING_ENABLED = process.env.NODE_ENV !== 'production' || window.__PROFILE_ENABLED__;

function ConditionalProfiler({
  id,
  children,
}: {
  id: string;
  children: ReactNode;
}): ReactElement {
  if (PROFILING_ENABLED) {
    return (
      <Profiler id={id} onRender={onRender}>
        {children}
      </Profiler>
    );
  }
  return <>{children}</>;
}

// Production: query parameter ?profile=true
// window.__PROFILE_ENABLED__ = window.location.search.includes('profile=true');
```

</details>

---

## onRender Callback Fields

### Nazariya

`onRender` callback — `<Profiler>`'ning ish maydoni:

```typescript
type ProfilerOnRenderCallback = (
  id: string,
  phase: 'mount' | 'update' | 'nested-update',
  actualDuration: number,
  baseDuration: number,
  startTime: number,
  commitTime: number
) => void;
```

### Fields

#### 1. `id`

Profiler identifier — `<Profiler id="X">` prop'idan kelgan string.

```typescript
[App] mount: 5.2ms
[Header] mount: 0.5ms
```

Use: granular monitoring (har Profiler alohida metric).

#### 2. `phase`

Render phase:

- **`mount`** — initial render (komponent yangi).
- **`update`** — re-render (state/props change).
- **`nested-update`** (R18+) — bu commit oldingi commit'ning `useLayoutEffect` yoki `useEffect` ichidagi `setState` natijasida boshlangan. Anti-pattern emas — DOM measure'dan keyin state set (masalan, tooltip position'ini layout'dan o'qib state'ga yozish) standart pattern.

Use: mount vs update vs nested-update statistik distinguish (nested-update tez-tez bo'lsa — measure-then-update pattern qayta-qayta layout shift keltirib chiqishi mumkin, optimization candidate).

#### 3. `actualDuration`

Real ish vaqti milliseconds — `<Profiler>` subtree shu commit'da render qilingan vaqt:

```typescript
actualDuration = sum of (fiber render time for fibers that rendered in this commit)
```

Bailout fibers `actualDuration` ga qo'shilmaydi.

Misol:

```
Commit:
   App rendered (1ms)
   Header bailout (0ms)
   MainContent rendered (10ms)
   
Profiler "App": actualDuration = 1 + 10 = 11ms (Header skipped)
```

#### 4. `baseDuration`

**Memoization'siz** rendering vaqti — agar barcha komponent'lar re-render bo'lsa:

```typescript
baseDuration = sum of (selfBaseDuration for all fibers in subtree)
```

Bailout va memo e'tiborga olinmaydi — har fiber sanaladi.

Memoization efficiency formula:

```
Efficiency = (baseDuration - actualDuration) / baseDuration × 100%

Misol:
   baseDuration = 50ms (hammasi rendered)
   actualDuration = 5ms (90% bailout)
   Efficiency = 90%
```

100% efficiency — barcha komponent'lar memo bailout. 0% — hech qanday memoization.

#### 5. `startTime`

Render boshlangan vaqt (`performance.now()` timestamp):

```typescript
startTime: 12345.67  // millisecond from page load
```

Use: timing correlation (other events bilan).

#### 6. `commitTime`

Commit Phase boshlangan vaqt:

```typescript
commitTime: 12356.89
```

Render → Commit gap:

```typescript
const renderToCommitGap = commitTime - startTime - actualDuration;
// Time between render done and commit start
// Usually < 1ms (Scheduler overhead)
```

### Real-World Patterns

```typescript
const onRender: ProfilerOnRenderCallback = (
  id,
  phase,
  actualDuration,
  baseDuration,
  startTime,
  commitTime
) => {
  // 1. Slow render detection
  if (actualDuration > 50) {
    console.warn(`[Slow] ${id} ${phase}: ${actualDuration}ms`);
  }
  
  // 2. Memoization efficiency
  const efficiency = ((baseDuration - actualDuration) / baseDuration) * 100;
  if (efficiency < 30) {
    console.warn(`[Low memo] ${id}: ${efficiency.toFixed(0)}% bailout`);
  }
  
  // 3. Production telemetry
  monitoring.recordReactRender({
    id,
    phase,
    actualDuration,
    baseDuration,
    timestamp: commitTime,
  });
};
```

<details>
<summary><strong>Under the Hood</strong></summary>

Field calculation in React:

```javascript
// react-dom/profiling
function commitWork(workInProgress) {
  // ... commit DOM
  
  if (workInProgress.tag === Profiler) {
    const onRender = workInProgress.memoizedProps.onRender;
    const id = workInProgress.memoizedProps.id;
    
    onRender(
      id,
      workInProgress.alternate === null ? 'mount' : 'update',
      workInProgress.actualDuration,
      workInProgress.treeBaseDuration,
      workInProgress.actualStartTime,
      currentCommitTime
    );
  }
  
  // ...
}
```

`actualDuration` Reconciler tomonidan accumulated:

```javascript
function performUnitOfWork(workInProgress) {
  const startTime = now();
  const next = beginWork(workInProgress);
  const elapsed = now() - startTime;
  
  workInProgress.actualDuration += elapsed;
  
  // Propagate to ancestors
  let parent = workInProgress.return;
  while (parent !== null) {
    if (parent.tag === Profiler) {
      parent.actualDuration += elapsed;
    }
    parent = parent.return;
  }
  
  return next;
}
```

`treeBaseDuration` — subtree'dagi har fiber'ning `selfBaseDuration` qiymatini yig'ish (oxirgi render'dan o'lchangan). Bashorat emas, balki memoization olib tashlangan holatdagi nazariy tree render vaqti:

```javascript
function completeWork(workInProgress) {
  // ...
  // Har fiber'ning oxirgi render'da o'lchangan o'z selfBaseDuration'idan boshlanadi
  workInProgress.treeBaseDuration = workInProgress.selfBaseDuration;
  
  // Children'ning aggregat treeBaseDuration'i qo'shiladi
  let child = workInProgress.child;
  while (child !== null) {
    workInProgress.treeBaseDuration += child.treeBaseDuration;
    child = child.sibling;
  }
}
```

`selfBaseDuration` — har fiber render qilinganda update qilinadi (memoization bailout bo'lsa, oldingi qiymat reused). Shu sababli `treeBaseDuration` bailout fiber'larni ham hisobga oladi (eski o'lchov qiymati bilan) — bu "no-memo" baseline.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Comprehensive monitoring:

```tsx
import { Profiler } from 'react';
import type { ReactElement, ProfilerOnRenderCallback, ReactNode } from 'react';

interface RenderMetric {
  id: string;
  phase: string;
  actualDuration: number;
  baseDuration: number;
  efficiency: number;
  startTime: number;
  commitTime: number;
  timestamp: number;
}

class PerformanceMonitor {
  private metrics: RenderMetric[] = [];
  private slowRenderThreshold = 16; // 1 frame at 60fps
  
  record: ProfilerOnRenderCallback = (
    id,
    phase,
    actualDuration,
    baseDuration,
    startTime,
    commitTime
  ) => {
    const efficiency =
      baseDuration > 0
        ? ((baseDuration - actualDuration) / baseDuration) * 100
        : 0;
    
    const metric: RenderMetric = {
      id,
      phase,
      actualDuration,
      baseDuration,
      efficiency,
      startTime,
      commitTime,
      timestamp: Date.now(),
    };
    
    this.metrics.push(metric);
    
    // Slow render alert
    if (actualDuration > this.slowRenderThreshold) {
      console.warn(`[Slow render] ${id} ${phase}: ${actualDuration.toFixed(2)}ms`);
    }
    
    // Memory cap
    if (this.metrics.length > 1000) {
      this.flush();
    }
  };
  
  flush() {
    if (this.metrics.length === 0) return;
    
    const batch = this.metrics.splice(0, this.metrics.length);
    
    // Send via Beacon API (no response wait)
    navigator.sendBeacon(
      '/api/monitoring/react',
      JSON.stringify(batch)
    );
  }
  
  getStats(componentId?: string): {
    count: number;
    avgDuration: number;
    p95Duration: number;
    avgEfficiency: number;
  } {
    const filtered = componentId
      ? this.metrics.filter((m) => m.id === componentId)
      : this.metrics;
    
    if (filtered.length === 0) {
      return { count: 0, avgDuration: 0, p95Duration: 0, avgEfficiency: 0 };
    }
    
    const durations = filtered.map((m) => m.actualDuration).sort((a, b) => a - b);
    const p95Index = Math.floor(durations.length * 0.95);
    
    return {
      count: filtered.length,
      avgDuration: durations.reduce((s, d) => s + d, 0) / durations.length,
      p95Duration: durations[p95Index],
      avgEfficiency:
        filtered.reduce((s, m) => s + m.efficiency, 0) / filtered.length,
    };
  }
}

const monitor = new PerformanceMonitor();

// Periodic flush
setInterval(() => monitor.flush(), 10000);

// Cleanup on unload
window.addEventListener('beforeunload', () => monitor.flush());

interface MonitoredAppProps {
  children: ReactNode;
}

function MonitoredApp({ children }: MonitoredAppProps): ReactElement {
  return (
    <Profiler id="App" onRender={monitor.record}>
      {children}
    </Profiler>
  );
}

// Debug stats (console)
// monitor.getStats('App') →
// { count: 100, avgDuration: 5.2, p95Duration: 25.4, avgEfficiency: 75 }
```

</details>

---

## Production Profiling — `react-dom/profiling`

### Nazariya

Production build'da `<Profiler>` default **no-op** (overhead minimal). Profile data measure qilish uchun **profiling build** kerak.

`react-dom/profiling` — alternative production build, Profiler API enabled.

### Setup

#### Vite

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      'react-dom$': 'react-dom/profiling',
    },
  },
});
```

> **Eslatma:** Eski tutorial'larda `'scheduler/tracing': 'scheduler/tracing-profiling'` alias uchratish mumkin — bu `interactions` tracing API uchun edi (`onRender` callback'ning 7-argumenti). React 17'da deprecated, React 18+'da olib tashlangan. Modern setup'da kerak emas.

#### Webpack

```javascript
// webpack.config.js
module.exports = {
  resolve: {
    alias: {
      'react-dom$': 'react-dom/profiling',
    },
  },
};
```

#### Next.js

```javascript
// next.config.js
module.exports = {
  webpack: (config, { isServer }) => {
    if (!isServer) {
      config.resolve.alias = {
        ...config.resolve.alias,
        'react-dom$': 'react-dom/profiling',
      };
    }
    return config;
  },
};
```

### Build Comparison

| Build | Profiler API | DevTools support | Bundle size | Use case |
|-------|--------------|------------------|-------------|----------|
| Development | ✅ | ✅ | Largest (DEV warnings) | Local dev |
| Production | ❌ no-op | Limited | Smallest | Production deploy |
| Production + profiling | ✅ | ✅ | Slightly larger | Production monitoring |

Bundle hajmi farqi:

`react-dom/profiling` build production minified, lekin Profiler instrumentation (commitProfiler hook'lar, actualDuration/treeBaseDuration tracking kod yo'llari) saqlanadi — bu standart `react-dom.production.min.js`'dan birmuncha kattaroq bundle beradi. Aniq farq React versiyasi va build flag'larga bog'liq (rasmiy raqam manbasi — qo'llanilayotgan React versiyasi `package.json` va bundler stats).

Trade-off — qo'shimcha bundle hajmi vs. production profile capability. Sampling deployment (faqat user'lar foizi profiling build oladi) bu xarajatni minimize qiladi.

### Conditional Profiling Build

Faqat ba'zi user'larda profiling build (sampling):

```typescript
// vite.config.ts
const PROFILING_ENABLED = process.env.PROFILING === 'true';

export default defineConfig({
  resolve: {
    alias: PROFILING_ENABLED
      ? {
          'react-dom$': 'react-dom/profiling',
        }
      : {},
  },
});
```

Build commands:

```bash
# Normal production
npm run build

# Profiling production
PROFILING=true npm run build
```

Deploy 1% user'larga profiling build (CDN sampling), monitoring data collect.

### DevTools Production Connection

Profiling build deploy qilingach, production'da React DevTools ishlaydi (development'dagi kabi). Lekin:

- Source map kerak (component names ko'rinishi uchun).
- DEV warning'lar yo'q (production behavior).
- Strict Mode 2x render production'da yo'q (real measure).

> **Eslatma:** Production'da DevTools faqat **trusted environment** (internal staging, debug session). Public app'da production source map exposed bo'lsa — security risk (kod logic ko'rinadi).

<details>
<summary><strong>Under the Hood</strong></summary>

`react-dom/profiling` package structure:

```
node_modules/react-dom/
├── profiling.js           ← Entry, includes profiler runtime
├── cjs/
│   ├── react-dom.development.js
│   ├── react-dom.production.min.js
│   └── react-dom.profiling.min.js   ← Production + Profiler API
```

Profiling build minified, lekin Profiler instrumentation saqlanadi:

```javascript
// react-dom.profiling.min.js (oddiylashtirilgan)
function commitWork(fiber) {
  // ...
  if (fiber.tag === Profiler) {
    // Profiler API enabled
    const onRender = fiber.memoizedProps.onRender;
    if (typeof onRender === 'function') {
      onRender(/* ... */);
    }
  }
  // ...
}

// vs production build (react-dom.production.min.js)
function commitWork(fiber) {
  // ...
  // Profiler Fiber tag tanildi, lekin instrumentation kod yo'li
  // bundle'ga kirmaydi (dead code elimination): timing field'lar
  // hisoblanmaydi, onRender chaqirilmaydi
  // ...
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Profile sampling strategy:

```typescript
// Production deployment with sampling
// Aksariyat users — normal production
// Kichik foiz (1%) users — profiling production (collect data)

// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

const isProfilingBuild = process.env.PROFILING === 'true';

export default defineConfig({
  plugins: [react()],
  resolve: isProfilingBuild
    ? {
        alias: {
          'react-dom$': 'react-dom/profiling',
        },
      }
    : {},
  build: {
    outDir: isProfilingBuild ? 'dist-profiling' : 'dist',
  },
});
```

```bash
# Build both
npm run build                    # → dist/
PROFILING=true npm run build     # → dist-profiling/
```

```nginx
# nginx — sampling routing
upstream backend_normal { server app:3000; }
upstream backend_profiling { server app-profiling:3001; }

server {
  location / {
    set $sample 1;
    if ($cookie_user_id ~ "^[0-9]") {
      # User ID ends with 0-9 — 10% chance
      set $first_digit $1;
      if ($first_digit ~ "[0]") {
        # 1% sample — profiling build
        proxy_pass http://backend_profiling;
        break;
      }
    }
    proxy_pass http://backend_normal;
  }
}
```

</details>

---

## Web Vitals Integration

### Nazariya

**Web Vitals** — Google'ning user-centric performance metrics. Real user experience'ga to'g'ridan-to'g'ri ta'sir qiladi.

### Core Web Vitals (2024-2026)

| Metric | Description | Target |
|--------|-------------|--------|
| **LCP** (Largest Contentful Paint) | Eng katta content render bo'lgan vaqt | < 2.5s |
| **INP** (Interaction to Next Paint) | User interaction → paint vaqt | < 200ms |
| **CLS** (Cumulative Layout Shift) | Layout shift score | < 0.1 |

> **Versiya evolyutsiyasi (Web Vitals):**
> - **2020 (May):** Web Vitals va Core Web Vitals e'lon qilindi (LCP, FID, CLS — Core; FCP, TTFB — qo'shimcha).
> - **2021 (June):** Core Web Vitals Google Search ranking signal'iga aylandi (Page Experience update).
> - **2024 (March):** **INP** stable bo'ldi va FID o'rniga **Core Web Vitals'ga qo'shildi**; FID Core'dan olib tashlandi (`web-vitals` v4'da `onFID` olib tashlangan).
> - **Sabab:** INP har user interaction'ning to'liq response vaqtini o'lchaydi (handler + paint + next frame), FID faqat birinchi input event'ning processing'gacha bo'lgan delay'ni o'lchardi — interaction lifecycle'ning faqat bir qismi.

### React'da Web Vitals

`web-vitals` package — Google rasmiy library:

```bash
npm install web-vitals
```

```typescript
import { onCLS, onINP, onLCP } from 'web-vitals';

onLCP((metric) => {
  console.log('LCP:', metric.value);
  monitoring.send({ name: 'LCP', value: metric.value });
});

onINP((metric) => {
  console.log('INP:', metric.value);
  monitoring.send({ name: 'INP', value: metric.value });
});

onCLS((metric) => {
  console.log('CLS:', metric.value);
  monitoring.send({ name: 'CLS', value: metric.value });
});
```

### React Profiler bilan Korrelyatsiya

Web Vitals (browser-level) va React Profiler (component-level) data'sini birga analiz qilish:

```typescript
import { Profiler } from 'react';
import { onINP } from 'web-vitals';

// React render data
const reactMetrics: any[] = [];

const onRender = (id, phase, actualDuration) => {
  reactMetrics.push({ id, phase, actualDuration, timestamp: performance.now() });
};

// INP data
onINP((metric) => {
  // metric.entries — interaction events
  const interactionTime = metric.entries[0]?.startTime;
  const inpValue = metric.value;
  
  // Find React renders during this interaction
  const relatedRenders = reactMetrics.filter(
    (m) => m.timestamp >= interactionTime && m.timestamp <= interactionTime + inpValue
  );
  
  console.log(`INP ${inpValue}ms — React renders during interaction:`);
  relatedRenders.forEach((m) => console.log(`  ${m.id}: ${m.actualDuration}ms`));
  
  // Slow INP → identify slow React component
  if (inpValue > 200) {
    monitoring.send({
      type: 'slow-inp',
      inp: inpValue,
      reactRenders: relatedRenders,
    });
  }
});
```

### LCP Optimization

LCP — eng katta content (image, video, large text). React'da:

1. **SSR/SSG** — server'da HTML pre-render, client'da darhol ko'rinadi.
2. **Lazy load below-fold** — `<img loading="lazy">` yoki React.lazy (cross-ref `35-code-splitting.md`).
3. **Image optimization** — Next.js `<Image>`, `srcset`, modern formats (AVIF, WebP).
4. **Critical CSS inline** — above-fold CSS HTML ichida.
5. **R19 Document Metadata** — `<link rel="preload">` JSX'da (cross-ref `37-react-19-document-apis.md`).

### INP Optimization

INP — user interaction (click, type) → paint. React'da:

1. **`useTransition`** — non-urgent updates Concurrent rendering (cross-ref `22-concurrent-hooks.md`).
2. **Component virtualization** — large list (cross-ref `36-virtualization.md`).
3. **Memoization** — re-render scope minimal.
4. **Event handler optimization** — heavy work `requestIdleCallback` ga move.

### CLS Optimization

CLS — layout shift (image, font, dynamic content). React'da:

1. **Image dimensions** — `width`/`height` attribute majburiy.
2. **Skeleton placeholders** — loading state'da fixed-size skeleton.
3. **Font loading** — `font-display: swap` yoki `optional`.
4. **Dynamic content reserve space** — `min-height` placeholder.
5. **Suspense fallback** — fixed-size loading state (cross-ref `29-suspense-lazy.md`).

<details>
<summary><strong>Under the Hood</strong></summary>

`web-vitals` package mexanika:

```javascript
// onLCP implementation (oddiylashtirilgan)
function onLCP(callback) {
  let largestEntry = null;
  
  const observer = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      if (!largestEntry || entry.startTime > largestEntry.startTime) {
        largestEntry = entry;
      }
    }
  });
  
  observer.observe({ type: 'largest-contentful-paint', buffered: true });
  
  // Report on page hide / unload
  document.addEventListener('visibilitychange', () => {
    if (document.visibilityState === 'hidden' && largestEntry) {
      callback({
        name: 'LCP',
        value: largestEntry.renderTime || largestEntry.loadTime,
        entries: [largestEntry],
      });
    }
  });
}
```

`PerformanceObserver` browser API — performance entries'ni async track qiladi.

INP measurement (`web-vitals` v4 algoritm, oddiylashtirilgan):

```javascript
// Browser native: 'event' Performance API har interaction'ning to'liq
// duration'ini (input handler + render + paint) o'lchaydi
function onINP(callback) {
  // Eng sekin 10 interaction'ni saqlash (reservoir)
  const slowestInteractions = []; // sorted descending by duration
  const MAX_RESERVOIR = 10;
  
  const observer = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      // entry.duration — interaction'ning to'liq lifecycle vaqti
      const interaction = { duration: entry.duration, entry };
      
      // Reservoir'ga qo'shish (eng sekin 10'ta saqlanadi)
      slowestInteractions.push(interaction);
      slowestInteractions.sort((a, b) => b.duration - a.duration);
      if (slowestInteractions.length > MAX_RESERVOIR) {
        slowestInteractions.length = MAX_RESERVOIR;
      }
    }
  });
  observer.observe({ type: 'event', durationThreshold: 40, buffered: true });
  
  // INP qiymatini hisoblash:
  // - 50+ interaction bo'lsa: ~98-th percentile (har 50 interaction'da 1'ta drop)
  // - 50'dan kam: eng sekin interaction
  function computeINP(totalInteractionCount) {
    if (slowestInteractions.length === 0) return null;
    // Har 50 interaction'da 1'ta drop — index = floor(count / 50)
    const dropIndex = Math.min(
      Math.floor(totalInteractionCount / 50),
      slowestInteractions.length - 1
    );
    return slowestInteractions[dropIndex];
  }
  
  // Report page hide/unload'da (NOT setInterval — final value page session uchun)
  document.addEventListener('visibilitychange', () => {
    if (document.visibilityState === 'hidden') {
      const inp = computeINP(/* total count */);
      if (inp) callback({ name: 'INP', value: inp.duration, entries: [inp.entry] });
    }
  });
}
```

INP — page session uchun **eng yomon** interaction (>50 interaction bo'lsa, top 2% drop bilan). Boshqa Web Vitals (LCP, CLS) ham final value page hide'da reported — bu sababli `navigator.sendBeacon` ishlatish kerak (sync, unload kafolatlangan).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Production Web Vitals + Profiler integration:

```tsx
import { useEffect } from 'react';
import { Profiler } from 'react';
import { onCLS, onINP, onLCP, onFCP, onTTFB } from 'web-vitals';
import type { ReactElement, ReactNode, ProfilerOnRenderCallback } from 'react';

interface VitalsMetric {
  name: 'LCP' | 'INP' | 'CLS' | 'FCP' | 'TTFB';
  value: number;
  rating: 'good' | 'needs-improvement' | 'poor';
}

// Web Vitals reporter
function reportVitals(metric: VitalsMetric) {
  console.log(`[Vitals] ${metric.name}: ${metric.value}ms (${metric.rating})`);
  
  navigator.sendBeacon(
    '/api/monitoring/vitals',
    JSON.stringify(metric)
  );
}

// Initialize Web Vitals
function initWebVitals(): void {
  onLCP((metric) => reportVitals({
    name: 'LCP',
    value: metric.value,
    rating: metric.rating,
  }));
  
  onINP((metric) => reportVitals({
    name: 'INP',
    value: metric.value,
    rating: metric.rating,
  }));
  
  onCLS((metric) => reportVitals({
    name: 'CLS',
    value: metric.value,
    rating: metric.rating,
  }));
  
  onFCP((metric) => reportVitals({
    name: 'FCP',
    value: metric.value,
    rating: metric.rating,
  }));
  
  onTTFB((metric) => reportVitals({
    name: 'TTFB',
    value: metric.value,
    rating: metric.rating,
  }));
}

// React render data
const renderHistory: { id: string; duration: number; timestamp: number }[] = [];

const onReactRender: ProfilerOnRenderCallback = (id, phase, actualDuration) => {
  renderHistory.push({
    id,
    duration: actualDuration,
    timestamp: performance.now(),
  });
  
  // Cap memory
  if (renderHistory.length > 1000) {
    renderHistory.shift();
  }
};

interface AppProps {
  children: ReactNode;
}

function App({ children }: AppProps): ReactElement {
  useEffect(() => {
    initWebVitals();
  }, []);
  
  return (
    <Profiler id="App" onRender={onReactRender}>
      {children}
    </Profiler>
  );
}

// INP correlation: slow INP → find React renders during interaction
// (onINP allaqachon yuqorida import qilingan)

onINP((metric) => {
  if (metric.value > 200) {
    const interactionStart = metric.entries[0]?.startTime ?? 0;
    const interactionEnd = interactionStart + metric.value;
    
    const slowReactRenders = renderHistory.filter(
      (r) =>
        r.timestamp >= interactionStart &&
        r.timestamp <= interactionEnd &&
        r.duration > 50
    );
    
    if (slowReactRenders.length > 0) {
      console.warn('Slow INP correlated with React renders:', slowReactRenders);
      
      navigator.sendBeacon(
        '/api/monitoring/slow-inp-react',
        JSON.stringify({ inp: metric.value, renders: slowReactRenders })
      );
    }
  }
});
```

</details>

---

## Real-World Workflow

### Nazariya

Production performance debugging — strukturlashgan jarayon. 8 qadam:

### Qadam 1: Symptom Identification

User report yoki monitoring alert:

```
- "Dashboard loading slow"
- "Search input laggy"
- "p95 commit duration > 100ms (last 24h)"
- "INP > 500ms for 5% users"
```

### Qadam 2: Reproduce

Stepslarni aniqlash:

```
1. Login as user with 1000+ records
2. Navigate to /dashboard
3. Type in search box
4. Observe lag
```

Conditions:

- User segment (low-end device, slow network)?
- Browser specific?
- Time of day (server load)?

### Qadam 3: Local Reproduce

Stepslarni local'da takrorlash:

```bash
# Mock 1000 records
curl -X POST localhost:3000/api/seed -d '{"count": 1000}'

# Browser:
# - Chrome DevTools → Performance → CPU 4x slowdown
# - Network → Fast 3G
# - Reproduce steps
```

### Qadam 4: Profile Record

React DevTools Profiler:

```
1. F12 → Profiler tab
2. Settings → ☑ "Record why each component rendered"
3. Click record (circle)
4. Reproduce stepslar
5. Stop record
```

### Qadam 5: Analyze

Flame chart inspect:

- **Wide bars** — bottleneck candidates.
- **Frequent commits** — re-render churn.
- **Wrong reasons** — "Why did this render?" panel'da.

Common patterns:

- Inline objects → `useMemo` / Compiler.
- Inline functions → `useCallback` / Compiler.
- Context propagation → split / selector.
- Large list → virtualization.

### Qadam 6: Hypothesis va Optimize

Hypothesis: "ItemList re-renders on every keystroke because of inline filter array."

Apply optimization:

```tsx
// Before
<ItemList items={data.filter((d) => d.type === filter)} />

// After
const filteredItems = useMemo(
  () => data.filter((d) => d.type === filter),
  [data, filter]
);
<ItemList items={filteredItems} />
```

### Qadam 7: Re-Profile va Confirm

Profile qaytadan, improvement aniqlash:

```
Before:
   ItemList: 50ms (red, slow)
   Why: Props changed (items)

After:
   ItemList: 0.5ms (green, fast)
   Why: Props changed (items)  ← still, but bailout in children
   ItemList children: bailout
```

### Qadam 8: Production Deploy + Monitoring

Deploy va monitoring:

```typescript
// Production telemetry
const onRender = (id, phase, actualDuration) => {
  if (actualDuration > 50) {
    monitoring.send({ type: 'slow-react', id, duration: actualDuration });
  }
};

// Web Vitals
onINP((metric) => {
  if (metric.value > 200) {
    monitoring.send({ type: 'slow-inp', value: metric.value });
  }
});
```

Trend monitoring:

- Daily p95 commit duration trend.
- INP distribution histogram.
- Component-specific outliers.

Alert:

- p95 > 50ms → PagerDuty notify.
- INP > 200ms for 10%+ users → Slack alert.

<details>
<summary><strong>Under the Hood</strong></summary>

Optimization workflow as DAG:

```
[Symptom] → [Reproduce] → [Profile] → [Hypothesis] → [Apply] → [Re-Profile]
                                                                       │
                                  [Confirm] ◀──────────────────────────┘
                                       │
                                       ▼
                              ┌────────┴────────┐
                              │                 │
                    [Improved] [Not improved]
                              │                 │
                              ▼                 ▼
                  [Deploy + Monitor] [Try different hypothesis]
```

Iteration cycle:

- 1 hypothesis: ~30-60 min profile + apply + re-profile.
- 3-5 iterations typical for complex bug.
- Don't apply 5 changes at once — measure each individually.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Real-world session simulation:

```tsx
// === Symptom: SearchBox laggy ===
import { useState, useMemo, useCallback, memo, Profiler } from 'react';
import type { ReactElement, ProfilerOnRenderCallback } from 'react';

interface Product {
  id: string;
  name: string;
  category: string;
  price: number;
}

// === Step 1-2: Initial implementation ===
function SearchBoxBefore({ products }: { products: Product[] }): ReactElement {
  const [query, setQuery] = useState('');
  
  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search products..."
      />
      <ProductList
        items={products.filter((p) => p.name.toLowerCase().includes(query.toLowerCase()))}
      />
    </div>
  );
}

function ProductList({ items }: { items: Product[] }): ReactElement {
  console.log('ProductList render', items.length);
  return (
    <ul>
      {items.map((p) => (
        <li key={p.id}>{p.name} — ${p.price}</li>
      ))}
    </ul>
  );
}

// === Step 3-4: Profile ===
// User report: typing laggy with 1000+ products
// Profile shows:
//   ProductList: 30-50ms per keystroke
//   Why: Props changed (items)
//   Bottleneck: filter executes per render, then map 1000 items

// === Step 5-6: Apply optimization ===
const ProductListMemo = memo(function ProductList({ items }: { items: Product[] }): ReactElement {
  return (
    <ul>
      {items.map((p) => (
        <li key={p.id}>{p.name} — ${p.price}</li>
      ))}
    </ul>
  );
});

function SearchBoxAfter({ products }: { products: Product[] }): ReactElement {
  const [query, setQuery] = useState('');
  
  // Memoize filtered items
  const filtered = useMemo(
    () =>
      products.filter((p) =>
        p.name.toLowerCase().includes(query.toLowerCase())
      ),
    [products, query]
  );
  
  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search products..."
      />
      <ProductListMemo items={filtered} />
    </div>
  );
}

// === Step 7: Re-profile ===
// SearchBox: 1ms (input update)
// ProductListMemo: 8-15ms (still render, but only when filter changes)
// Better, but still slow with 1000 items

// === Step 8: Further optimization — useTransition ===
import { useTransition } from 'react';

function SearchBoxFinal({ products }: { products: Product[] }): ReactElement {
  const [query, setQuery] = useState('');
  const [deferredQuery, setDeferredQuery] = useState('');
  const [isPending, startTransition] = useTransition();
  
  function handleChange(value: string) {
    setQuery(value); // Sync — input responsive
    startTransition(() => {
      setDeferredQuery(value); // Non-urgent — Transition Lane
    });
  }
  
  const filtered = useMemo(
    () =>
      products.filter((p) =>
        p.name.toLowerCase().includes(deferredQuery.toLowerCase())
      ),
    [products, deferredQuery]
  );
  
  return (
    <div>
      <input
        value={query}
        onChange={(e) => handleChange(e.target.value)}
      />
      {isPending && <span>Filtering...</span>}
      <div style={{ opacity: isPending ? 0.5 : 1 }}>
        <ProductListMemo items={filtered} />
      </div>
    </div>
  );
}

// === Step 8 continued: Production monitoring ===
const onRender: ProfilerOnRenderCallback = (id, phase, actualDuration) => {
  if (actualDuration > 50) {
    fetch('/api/monitoring/slow-render', {
      method: 'POST',
      body: JSON.stringify({ id, phase, actualDuration, timestamp: Date.now() }),
    });
  }
};

function App({ products }: { products: Product[] }): ReactElement {
  return (
    <Profiler id="SearchPage" onRender={onRender}>
      <SearchBoxFinal products={products} />
    </Profiler>
  );
}
```

</details>

---

## Highlight Updates Option

### Nazariya

**Highlight Updates** — DevTools Profiler tab'dagi visual debug feature. Har render bo'lgan komponent atrofida bir lahzaga rectangle ko'rinadi.

### Settings Yoqish

```
F12 → Profiler tab → Settings (gear) → ☑ "Highlight updates when components render"
```

### Color Coding

DevTools komponent'ning yaqin vaqt ichidagi re-render **chastotasi**'ni kuzatadi (recent commits rolling window) va shu chastotaga qarab rang tanlaydi:

| Border rangi | Ma'no |
|--------------|-------|
| Cool (yashil/teal) | Kam-uchraydigan re-render — yangi mount yoki ahyon-ahyonda update |
| Sariq | Moderate frequency — yaqin commit'larda bir necha re-render |
| Qizil | High frequency — tez ketma-ket ko'p re-render, hot spot, optimization candidate |

> **MUHIM:** Rang **commit chastotasi**ga bog'liq, ms duration emas. Strict Mode 2x cycle bitta commit ichida komponent funksiyasini ikki marta chaqiradi (alohida highlight emas, alohida render commit emas) — Highlight Updates'ning ranglarini "Strict Mode signal" deb o'qish noto'g'ri. Strict Mode'ni aniqlash uchun "Why did this render?" panel'da `mount (1st pass)` / `mount (2nd pass)` belgilar yoki Profiler'ning `actualDuration` ikki barobar oshishini kuzatish kerak.

### Use Cases

#### Use Case 1: Visual Re-Render Tracking

Komponent qachon va qaerda re-render bo'layotganini ko'rish:

- Click button → qaysi rectangle yashil bo'ladi?
- Type input → barcha komponent'lar qizil bo'lyaptimi?

#### Use Case 2: Unexpected Re-Render

```
Type in SearchBox → expected ProductList re-render
But: Sidebar ham yashil → unexpected!
Investigate: Sidebar nima uchun re-render bo'lyapti?
```

#### Use Case 3: Performance Hotspot

Qizil rectangle = tez-tez re-render = optimization candidate:

```
Animation runs → Player component qizil
But: nearby Volume component ham qizil → unnecessary re-render
Optimization: Volume component memo qilish
```

### Limitations

- **Production'da yo'q** — DevTools settings dev-only.
- **Visual noise** — barcha komponent'lar render bo'lsa screen overwhelmed.
- **No quantitative data** — duration ko'rinmaydi (faqat render fact).
- **Strict Mode 2x sariq** — production accurate emas.

### Highlight Updates vs Profile

| Feature | Highlight Updates | Profile (Flame Chart) |
|---------|-------------------|----------------------|
| Visual feedback | ✅ Real-time | ❌ Post-record |
| Quantitative | ❌ | ✅ |
| Production | ❌ | ✅ (with profiling build) |
| Use case | Quick scan | Deep analysis |

Workflow:

1. **Highlight Updates** — quick visual scan (find unexpected renders).
2. **Profile** — quantitative analysis (durations, reasons).

<details>
<summary><strong>Under the Hood</strong></summary>

Highlight Updates implementation:

```javascript
// DevTools script
function onCommitFiberRoot(rendererID, root) {
  if (settings.highlightUpdates) {
    const renderedFibers = collectRenderedFibers(root);
    
    for (const fiber of renderedFibers) {
      const node = fiber.stateNode;
      if (node instanceof HTMLElement) {
        highlightElement(node, getColor(fiber.renderCount));
      }
    }
  }
}

function highlightElement(element, color) {
  const rect = element.getBoundingClientRect();
  const overlay = document.createElement('div');
  overlay.style.cssText = `
    position: fixed;
    top: ${rect.top}px;
    left: ${rect.left}px;
    width: ${rect.width}px;
    height: ${rect.height}px;
    border: 2px solid ${color};
    pointer-events: none;
    z-index: 9999;
  `;
  document.body.appendChild(overlay);
  
  setTimeout(() => overlay.remove(), 100);
}

function getColor(recentRenderCount) {
  // recentRenderCount — komponent'ning rolling window'dagi (yaqin commit'lar)
  // re-render soni. Strict Mode 2x cycle bitta commit'da bo'ladi va
  // alohida count sifatida hisoblanmaydi.
  if (recentRenderCount <= 1) return '#5cb85c'; // Cool — kam-uchraydigan
  if (recentRenderCount <= 3) return '#f0ad4e'; // Moderate frequency
  return '#d9534f'; // Hot — frequent
}
```

Render count tracking — DevTools komponent'ning yaqin commit'lardagi re-render chastotasini rolling window'da kuzatadi (vaqt o'tgan sayin count decay qilinadi).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Demo komponent — Highlight Updates test:

```tsx
import { useState, memo } from 'react';
import type { ReactElement } from 'react';

const Sidebar = memo(function Sidebar(): ReactElement {
  return <aside>Sidebar (should be green only on mount)</aside>;
});

const FrequentUpdater = memo(function FrequentUpdater({ value }: { value: number }): ReactElement {
  return <div>Frequent: {value}</div>;
});

function App(): ReactElement {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <button onClick={() => setCount((c) => c + 1)}>
        Click ({count})
      </button>
      <Sidebar />
      <FrequentUpdater value={count} />
    </div>
  );
}

// Highlight Updates ON, click button rapidly:
// - button rectangle: yashil (initial), then turns yellow then red on rapid clicks
// - Sidebar: yashil only on mount (memo bailout har click'da)
// - FrequentUpdater: yashil → sariq → qizil (frequent updates)
//
// Visual signal: FrequentUpdater = optimization candidate
// Investigate: value prop o'zgarib turadi, expected
// vs Sidebar — props yo'q, memo correct bailout
```

</details>

---

## Strict Mode va Profile Impact

### Nazariya

`<StrictMode>` development'da render funksiyalarni 2x chaqiradi (cross-ref `30-concurrent-react.md`). Profile data shu sababdan production'dan farq qiladi.

### Strict Mode Behavior

Development:

```typescript
<StrictMode>
  <App />
</StrictMode>
```

- Komponent body 2x chaqiriladi.
- `useState` initializer 2x.
- `useMemo` factory 2x.
- `useReducer` reducer 2x.
- R18+ `useEffect` mount→cleanup→mount cycle.

Production:

- Strict Mode no-op (komponent 1x render).

### Profile Impact

Development profile:

```
Component render: 5ms (Strict Mode 2x = 2.5ms × 2)
Real production: 2.5ms
```

Profile data **Strict Mode bilan ~2x kattaroq** — actualDuration overestimate.

### Accurate Production Profile

Options:

#### Option 1: Strict Mode olib tashlash (development)

```typescript
// Profile session uchun
// <StrictMode> olib tashlash
const container = document.getElementById('root');
if (!container) throw new Error('Root container not found');
ReactDOM.createRoot(container).render(<App />);
```

Lekin Strict Mode bug surface qiladi — production deployment'gacha qaytarish.

#### Option 2: Production Build Profile

`react-dom/profiling` build ishlatish (yuqorida). Production'da Strict Mode no-op, accurate measurement.

#### Option 3: actualDuration Halve

Development data'ni 2'ga bo'lish:

```typescript
const onRender = (id, phase, actualDuration) => {
  const productionEstimate = process.env.NODE_ENV === 'development'
    ? actualDuration / 2 // Strict Mode tax
    : actualDuration;
  
  console.log(id, phase, productionEstimate);
};
```

Bu approximation — Strict Mode'ning real overhead production'dan slightly farq qiladi.

### Strict Mode Render Count

DevTools "Why did this render?" panel'da ko'rinadi:

```
Component re-rendered:
  • Mount (1st pass)
  • Mount (2nd pass — Strict Mode)
```

Yashil → sariq highlighting (Strict Mode 2x).

### useEffect Strict Mode 2x Cycle (R18+)

```typescript
useEffect(() => {
  console.log('Setup');
  return () => console.log('Cleanup');
}, []);

// Strict Mode 2x cycle:
// 1. Setup
// 2. Cleanup
// 3. Setup
// (3 logs instead of 1)
```

Profile'da effect duration ham 2x ko'rinadi. Real production: 1 setup.

### Best Practice

- **Local development** — `<StrictMode>` har doim yoqilgan (bug surface).
- **Profile session** — qisqa muddatga olib tashlash YOKI production build profile.
- **Production** — Strict Mode yo'q, real measurement.

<details>
<summary><strong>Under the Hood</strong></summary>

Strict Mode 2x render implementation:

```javascript
// React internal (DEV only)
function callComponentInDEV(Component, props) {
  let result = Component(props); // 1st call
  
  if (isStrictModeFiber(workInProgress)) {
    // 2nd call — purity check
    Component(props);
  }
  
  return result;
}
```

`useState` initializer:

```javascript
// mountState (DEV)
function mountState(initialState) {
  const hook = mountWorkInProgressHook();
  
  if (typeof initialState === 'function') {
    initialState = initialState(); // 1st call
    
    if (isStrictModeFiber()) {
      const initialStateAgain = initialState(); // 2nd call (DEV)
      // Compare results — purity violation warning
    }
  }
  
  hook.memoizedState = initialState;
  // ...
}
```

R18+ Effect 2x cycle:

```javascript
// commitHookEffectListMount (DEV)
function commitHookEffectListMount() {
  for (const effect of effectList) {
    if (effect.tag === HookPassive) {
      effect.destroy = effect.create();
      
      if (isStrictModeFiber() && isMount) {
        // Strict Mode mount cycle
        if (effect.destroy) {
          effect.destroy(); // Cleanup
        }
        effect.destroy = effect.create(); // Re-setup
      }
    }
  }
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Profile-friendly Strict Mode toggle:

```tsx
import { StrictMode } from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';

const PROFILE_MODE = window.location.search.includes('profile=true');

const container = document.getElementById('root');
if (!container) throw new Error('Root container not found');
const root = ReactDOM.createRoot(container);

if (PROFILE_MODE) {
  // Profile session — no Strict Mode
  root.render(<App />);
} else {
  // Development — Strict Mode for bug surface
  root.render(
    <StrictMode>
      <App />
    </StrictMode>
  );
}

// Usage:
// http://localhost:3000?profile=true → no Strict Mode (accurate profile)
// http://localhost:3000              → Strict Mode (development default)
```

Production-like profile in development:

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

const PROFILING = process.env.PROFILING === 'true';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: PROFILING
      ? {
          'react-dom$': 'react-dom/profiling',
        }
      : {},
  },
  define: {
    __PROFILING__: PROFILING,
    'process.env.NODE_ENV': JSON.stringify(PROFILING ? 'production' : 'development'),
  },
});
```

```bash
# Development with Strict Mode (bug surface)
npm run dev

# Production-like profile session
PROFILING=true npm run build && npm run preview
```

</details>

---

## Performance Budget va Alerting

### Nazariya

**Performance budget** — performance metric'lar uchun maximum allowed value. Budget oshib ketsa — alert/block.

### Budget Categories

#### React-Specific

```typescript
const reactBudget = {
  componentRenderP50: 5,    // ms (median)
  componentRenderP95: 16,   // ms (95th percentile, 1 frame)
  componentRenderP99: 50,   // ms (slow but acceptable)
  totalCommitP95: 30,       // ms
  reRenderFrequencyMax: 10, // per second
};
```

#### Web Vitals

```typescript
const vitalsBudget = {
  lcp: 2500,    // ms (good threshold)
  inp: 200,     // ms
  cls: 0.1,     // unitless score
  fcp: 1800,    // ms
  ttfb: 800,    // ms
};
```

#### Bundle Size

```typescript
const bundleBudget = {
  totalJS: 200_000,         // bytes (gzipped)
  initialChunk: 80_000,     // bytes
  totalCSS: 50_000,         // bytes
  totalImages: 1_000_000,   // bytes
};
```

### Alerting Setup

#### Real-Time Alerts

```typescript
const onRender = (id, phase, actualDuration) => {
  if (actualDuration > reactBudget.componentRenderP99) {
    monitoring.send({
      type: 'budget-violation',
      metric: 'componentRender',
      value: actualDuration,
      budget: reactBudget.componentRenderP99,
      component: id,
    });
  }
};
```

#### Aggregated Alerts

```typescript
// Server-side aggregation (Datadog, Prometheus)
const dailyMetrics = aggregateMetrics(rawData);

if (dailyMetrics.p95CommitDuration > reactBudget.totalCommitP95) {
  alert.send({
    severity: 'high',
    title: `p95 commit duration exceeded budget: ${dailyMetrics.p95CommitDuration}ms`,
    runbook: 'https://wiki/react-performance-runbook',
  });
}
```

#### CI/CD Integration

Build'da bundle size budget check:

```javascript
// vite.config.ts
import { defineConfig } from 'vite';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  build: {
    rollupOptions: {
      onwarn(warning, warn) {
        if (warning.code === 'BUNDLE_SIZE_LIMIT') {
          throw new Error(`Bundle size budget exceeded: ${warning.message}`);
        }
        warn(warning);
      },
    },
  },
});
```

Lighthouse CI:

```yaml
# .github/workflows/lighthouse.yml
- name: Run Lighthouse CI
  run: |
    npm install -g @lhci/cli
    lhci autorun --collect.url=https://staging.example.com
  env:
    LHCI_GITHUB_APP_TOKEN: ${{ secrets.LHCI_GITHUB_APP_TOKEN }}
```

`.lighthouserc.json`:

```json
{
  "ci": {
    "assert": {
      "assertions": {
        "largest-contentful-paint": ["error", { "maxNumericValue": 2500 }],
        "interaction-to-next-paint": ["error", { "maxNumericValue": 200 }],
        "cumulative-layout-shift": ["error", { "maxNumericValue": 0.1 }]
      }
    }
  }
}
```

### Trend Analysis

Daily/weekly trends — regression early detection:

```
Week 1: p95 = 25ms
Week 2: p95 = 28ms (+12%)
Week 3: p95 = 35ms (+25%)  ← Trend alert
Week 4: p95 = 50ms (+43%)  ← Critical alert
```

Tools:

- **Datadog** — APM, custom dashboards.
- **Sentry** — Performance monitoring, transactions.
- **New Relic** — Browser monitoring.
- **Custom** — Influx + Grafana, Prometheus + Alertmanager.

<details>
<summary><strong>Under the Hood</strong></summary>

Aggregation (server-side):

```typescript
function aggregateMetrics(rawData: RenderMetric[]): DailyStats {
  const grouped = groupBy(rawData, (m) => m.id);
  
  const stats: DailyStats = {};
  
  for (const [id, metrics] of grouped) {
    const durations = metrics.map((m) => m.actualDuration).sort((a, b) => a - b);
    
    stats[id] = {
      count: metrics.length,
      p50: percentile(durations, 0.5),
      p95: percentile(durations, 0.95),
      p99: percentile(durations, 0.99),
      max: durations[durations.length - 1],
      avg: durations.reduce((s, d) => s + d, 0) / durations.length,
    };
  }
  
  return stats;
}

function percentile(sorted: number[], p: number): number {
  return sorted[Math.floor(sorted.length * p)];
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Production budget enforcement:

```tsx
import { Profiler } from 'react';
import { onLCP, onINP, onCLS } from 'web-vitals';
import type { ReactElement, ReactNode, ProfilerOnRenderCallback } from 'react';

interface PerformanceBudget {
  reactRenderP99: number;
  webVitalsLCP: number;
  webVitalsINP: number;
  webVitalsCLS: number;
}

const BUDGET: PerformanceBudget = {
  reactRenderP99: 50,
  webVitalsLCP: 2500,
  webVitalsINP: 200,
  webVitalsCLS: 0.1,
};

interface BudgetViolation {
  type: 'react' | 'lcp' | 'inp' | 'cls';
  value: number;
  budget: number;
  componentId?: string;
  timestamp: number;
}

const violations: BudgetViolation[] = [];

function reportViolation(v: BudgetViolation) {
  violations.push(v);
  console.warn(`[Budget violation] ${v.type}: ${v.value} > ${v.budget}`);
  
  navigator.sendBeacon(
    '/api/monitoring/budget-violation',
    JSON.stringify(v)
  );
}

const onRender: ProfilerOnRenderCallback = (id, phase, actualDuration) => {
  if (actualDuration > BUDGET.reactRenderP99) {
    reportViolation({
      type: 'react',
      value: actualDuration,
      budget: BUDGET.reactRenderP99,
      componentId: id,
      timestamp: Date.now(),
    });
  }
};

// Web Vitals enforcement
function setupVitalsMonitoring() {
  onLCP((metric) => {
    if (metric.value > BUDGET.webVitalsLCP) {
      reportViolation({
        type: 'lcp',
        value: metric.value,
        budget: BUDGET.webVitalsLCP,
        timestamp: Date.now(),
      });
    }
  });
  
  onINP((metric) => {
    if (metric.value > BUDGET.webVitalsINP) {
      reportViolation({
        type: 'inp',
        value: metric.value,
        budget: BUDGET.webVitalsINP,
        timestamp: Date.now(),
      });
    }
  });
  
  onCLS((metric) => {
    if (metric.value > BUDGET.webVitalsCLS) {
      reportViolation({
        type: 'cls',
        value: metric.value,
        budget: BUDGET.webVitalsCLS,
        timestamp: Date.now(),
      });
    }
  });
}

interface MonitoredAppProps {
  children: ReactNode;
}

function MonitoredApp({ children }: MonitoredAppProps): ReactElement {
  // Setup once
  if (typeof window !== 'undefined' && !(window as any).__VITALS_INIT__) {
    setupVitalsMonitoring();
    (window as any).__VITALS_INIT__ = true;
  }
  
  return (
    <Profiler id="App" onRender={onRender}>
      {children}
    </Profiler>
  );
}
```

</details>

---

## Edge Cases va Gotchas

### Gotcha 1: Profiler Tab "No data" Bug

Recording'dan keyin Profiler tab "No profiling data" deb chiqaradi:

**Sabablar:**
- DevTools versiyasi React versiyasidan eski.
- Strict Mode bilan recording boshlanmagan.
- Settings'da "Record while profiling" disabled.

**Yechim:** DevTools update, settings tekshirish, page reload.

### Gotcha 2: actualDuration < 0

Rare bug — DevTools `actualDuration < 0` ko'rsatadi.

**Sabab:** `performance.now()` clock skew (ba'zi browser'larda).

**Yechim:** `Math.max(0, actualDuration)` sanitize.

### Gotcha 3: `onRender` Callback'da Cheksiz Accumulation

`<Profiler>` har commit'da `onRender` chaqiradi. Closure'da existing state'ga reference ushlash o'z-o'zidan leak emas (data baribir App state'da bor). Real leak — `onRender` ichida boundsiz buffer'ga push qilish:

```tsx
// ❌ Real leak — har commit'da metrics array o'sadi
const metrics: any[] = [];

const onRender = (id, phase, duration) => {
  metrics.push({ id, phase, duration, timestamp: Date.now() });
  // Hech qachon flush yoki bound yo'q → heap o'sadi
};

return <Profiler id="App" onRender={onRender}>...</Profiler>;
```

**Yechim:** Buffer'ga bound qo'yish va periodic flush:

```tsx
const MAX_BUFFER = 100;
const metrics: any[] = [];

const onRender = (id, phase, duration) => {
  metrics.push({ id, phase, duration });
  if (metrics.length >= MAX_BUFFER) {
    navigator.sendBeacon('/api/metrics', JSON.stringify(metrics));
    metrics.length = 0;
  }
};
```

### Gotcha 4: Production Profile Build Bundle Size

`react-dom/profiling` instrumentation kodi production build'dan birmuncha kattaroq (aniq qiymat React versiyasi va bundler tree-shaking'iga bog'liq — `webpack-bundle-analyzer` yoki `rollup-plugin-visualizer` bilan o'lchanadi). CDN traffic va mobile users uchun bu xarajat sezilarli bo'lishi mumkin.

**Yechim:** Sampling deployment — user'larning kichik foiziga (masalan 1%) profiling build, qolganiga normal build (CDN routing yoki feature flag).

### Gotcha 5: Web Vitals Page Unload Loss

Web Vitals callback page hide/unload'da chaqiriladi. Network request slow bo'lsa — data lost.

**Yechim:** `navigator.sendBeacon` ishlatish (sync, page unload'da kafolatlangan):

```typescript
function reportVital(metric) {
  navigator.sendBeacon('/api/vitals', JSON.stringify(metric));
}
```

---

## Common Mistakes

### ❌ Xato 1: Development Build Profile Production Hisoblash

```tsx
// Development profile: 50ms
// Conclusion: production ham 50ms — NOTO'G'RI
// Production: 25ms (Strict Mode 2x dev only)
```

**Yechim:** Production build profile (`react-dom/profiling`).

### ❌ Xato 2: `<Profiler>` Bilan Barcha Tree Wrap

```tsx
<Profiler id="A">
  <Profiler id="B">
    <Profiler id="C">
      <Profiler id="D">
        ...
      </Profiler>
    </Profiler>
  </Profiler>
</Profiler>
// Overhead — har commit'da 4 callback
```

**Yechim:** Strategic placement (top-level + suspected hot paths).

### ❌ Xato 3: Slow Render Without Root Cause

```
Symptom: Component slow
Action: Add useMemo everywhere
Result: Some improvement, but root cause ignored
```

**Yechim:** "Why did this render?" panel orqali root cause topish, targeted fix.

### ❌ Xato 4: DevTools Profile Recording'ni Uzoq Vaqt Yoniq Qoldirish

"Record why each component rendered" yoqilgan holatda DevTools har commit'da fiber tree snapshot + per-fiber render reason metadata saqlaydi. Uzun recording sessiyalarda DevTools tab heap'i sezilarli o'sadi, panel'ning o'zi sekinlashadi. Ba'zi developer'lar profile setting'ni yoqib qolib, kun davomida ishlashda davom etadi.

**Yechim:** Profile recording'ni faqat aniq scenario uchun (5-10 second), keyin to'xtatish. Recording oxirida settings'ni o'chirish opsional, lekin yangi profile session uchun yana yoqish kerak bo'ladi. Production deployment'ga `<Profiler>` instrumentation ham keraksiz overhead — sampling deployment ishlatish.

### ❌ Xato 5: Web Vitals Ignored

```
React Profiler optimization done.
But: LCP > 4s, INP > 500ms still slow.
Root cause: Network/loading, not React render.
```

**Yechim:** Web Vitals + React Profiler birga monitoring.

---

## Amaliy Mashqlar

### Mashq 1: Basic Profile Workflow (Oson)

DevTools Profiler bilan quyidagi komponentni profile qiling. Qaysi komponent eng sekin? "Why did this render?" qiymatlarini topib bering.

```tsx
import { useState } from 'react';
import type { ReactElement } from 'react';

function Counter(): ReactElement {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount((c) => c + 1)}>{count}</button>;
}

function Sidebar(): ReactElement {
  return <aside>Sidebar (no props, no state)</aside>;
}

function HeavyList(): ReactElement {
  // Imitatsiya: 1000 items
  const items = Array.from({ length: 1000 }, (_, i) => i);
  return (
    <ul>
      {items.map((i) => <li key={i}>Item {i}</li>)}
    </ul>
  );
}

function App(): ReactElement {
  return (
    <>
      <Counter />
      <Sidebar />
      <HeavyList />
    </>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

Profile workflow:

```
1. F12 → Profiler tab
2. Settings → ☑ "Record why each component rendered while profiling"
3. Click record (circle)
4. Click Counter button 5 times
5. Stop record
```

Expected results:

- **Initial mount commit:** barcha komponent'lar render qilinadi.
  - App: total tree time (heavy children'ni o'z ichiga oladi)
  - Counter: kichik render
  - Sidebar: kichik render
  - HeavyList: eng sekin — 1000 ta `<li>` ko'rsatish (initial mount)

- **Click commits (Counter button bosilganda):**
  - Counter: re-render — Hooks changed (`count` state)
  - App: **bailout** — App'da state/props o'zgarmagan, App funksiyasi qaytadan chaqirilmaydi
  - Sidebar: **bailout (skipped)** — App re-render bo'lmagani uchun Sidebar reconcile qilinmaydi (sibling lanes ham mos kelmaydi)
  - HeavyList: **bailout (skipped)** — xuddi Sidebar kabi, React faqat update lane bor bo'lgan subtree'ga kiradi

> **MUHIM:** Counter'ning setState'i Counter Fiber'iga lane mark qiladi. Reconciler root'dan boshlanadi: App'ning `lanes` o'zida update yo'q, lekin `childLanes` mavjud → React App'ni bailout qilib, lekin childLanes bor bo'lgan child'ga (Counter) kiradi. Sidebar va HeavyList esa o'z `childLanes`'ida update yo'q — React ularga umuman kirmaydi. Demak "Parent rendered" sababi bu yerda **qo'llanmaydi** — sibling state update top-down propagation keltirib chiqarmaydi.

"Why did this render?" panel (commit'da render bo'lgan komponent'lar uchun):

| Component | Reason (mount) | Reason (click) |
|-----------|---------------|----------------|
| App | First time | Render bo'lmaydi (bailout) |
| Counter | First time | Hooks changed (`count`) |
| Sidebar | First time | Render bo'lmaydi (skipped) |
| HeavyList | First time | Render bo'lmaydi (skipped) |

Click commit'lari **flame chart'da** Counter (va uning `<button>` ostidagi DOM update) ko'rinadi; Sidebar va HeavyList bar'lari gray (bailout) bo'ladi.

**Top-down propagation bug qachon yuzaga keladi:** agar Counter o'rniga **App'da** state bo'lsa (masalan, `const [count, setCount] = useState(0)` App ichida) va `count` boshqa joyda ishlatilsa. U holda App re-render qilinadi → Sidebar va HeavyList **Parent component rendered** sababi bilan re-render bo'ladi (memo'siz). Shu holatda `React.memo` ulardagi re-render'ni bailout qiladi:

```tsx
const Sidebar = memo(function Sidebar() {
  return <aside>Sidebar</aside>;
});

const HeavyList = memo(function HeavyList() {
  const items = Array.from({ length: 1000 }, (_, i) => i);
  return <ul>{items.map((i) => <li key={i}>Item {i}</li>)}</ul>;
});
```

Memo bilan Sidebar va HeavyList faqat props o'zgarganda re-render bo'ladi. Joriy misolda esa (state Counter ichida) memo umuman kerak emas — bailout React'ning standart fiber lanes mexanizmidan kelib chiqadi.

</details>

### Mashq 2: `<Profiler>` Callback Implementation (Oson)

`onRender` callback yarating: 3 kriteriya monitoring qiladi (slow render warning, memoization efficiency calc, monitoring service'ga yuborish).

```tsx
import { Profiler } from 'react';
import type { ReactElement, ProfilerOnRenderCallback, ReactNode } from 'react';

const onRender: ProfilerOnRenderCallback = (
  id,
  phase,
  actualDuration,
  baseDuration,
  startTime,
  commitTime
) => {
  // TODO: implement
};

function App({ children }: { children: ReactNode }): ReactElement {
  return (
    <Profiler id="App" onRender={onRender}>
      {children}
    </Profiler>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { Profiler } from 'react';
import type { ReactElement, ProfilerOnRenderCallback, ReactNode } from 'react';

const SLOW_RENDER_THRESHOLD = 16; // 1 frame at 60fps
const LOW_EFFICIENCY_THRESHOLD = 30; // %

interface MonitoringData {
  id: string;
  phase: string;
  actualDuration: number;
  baseDuration: number;
  efficiency: number;
  timestamp: number;
}

const buffer: MonitoringData[] = [];

const onRender: ProfilerOnRenderCallback = (
  id,
  phase,
  actualDuration,
  baseDuration,
  startTime,
  commitTime
) => {
  // Calc efficiency
  const efficiency =
    baseDuration > 0
      ? ((baseDuration - actualDuration) / baseDuration) * 100
      : 0;
  
  // 1. Slow render warning
  if (actualDuration > SLOW_RENDER_THRESHOLD) {
    console.warn(
      `[Slow render] ${id} ${phase}: ${actualDuration.toFixed(2)}ms`
    );
  }
  
  // 2. Low memoization efficiency warning
  if (baseDuration > 0 && efficiency < LOW_EFFICIENCY_THRESHOLD) {
    console.warn(
      `[Low memoization] ${id}: ${efficiency.toFixed(0)}% efficiency`
    );
  }
  
  // 3. Buffer for monitoring service
  buffer.push({
    id,
    phase,
    actualDuration,
    baseDuration,
    efficiency,
    timestamp: commitTime,
  });
  
  // Periodic flush
  if (buffer.length >= 50) {
    flush();
  }
};

function flush(): void {
  if (buffer.length === 0) return;
  
  const batch = buffer.splice(0, buffer.length);
  
  navigator.sendBeacon(
    '/api/monitoring/react',
    JSON.stringify(batch)
  );
}

// Periodic flush
setInterval(flush, 10000);

// Cleanup on unload
window.addEventListener('beforeunload', flush);

function App({ children }: { children: ReactNode }): ReactElement {
  return (
    <Profiler id="App" onRender={onRender}>
      {children}
    </Profiler>
  );
}
```

Test:

```tsx
// Slow render: 20ms render
// Console: "[Slow render] App update: 20.45ms"
// Buffer: { id: 'App', actualDuration: 20.45, ... }

// Low memoization: 50ms baseDuration, 45ms actualDuration → 10% efficiency
// Console: "[Low memoization] App: 10% efficiency"

// Every 10s yoki 50 commits:
// navigator.sendBeacon('/api/monitoring/react', batchData)
```

</details>

### Mashq 3: Production Profile Setup (O'rta)

Vite configuration yarating: profile build (`react-dom/profiling`) va normal build alohida output. CI'da deploy qachon profile build deploy qilinadi (1% sampling).

<details>
<summary><strong>Javob</strong></summary>

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

const isProfilingBuild = process.env.PROFILING === 'true';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: isProfilingBuild
      ? {
          'react-dom$': 'react-dom/profiling',
        }
      : {},
  },
  build: {
    outDir: isProfilingBuild ? 'dist-profiling' : 'dist',
    sourcemap: isProfilingBuild, // Profiling build needs source maps for DevTools
    rollupOptions: {
      output: {
        manualChunks: {
          react: ['react', 'react-dom'],
        },
      },
    },
  },
  define: {
    __PROFILING__: isProfilingBuild,
  },
});
```

```json
// package.json
{
  "scripts": {
    "build": "vite build",
    "build:profiling": "PROFILING=true vite build",
    "build:all": "npm run build && npm run build:profiling"
  }
}
```

CI/CD pipeline:

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Install
        run: npm ci
      
      - name: Build normal
        run: npm run build
      
      - name: Build profiling
        run: npm run build:profiling
      
      - name: Deploy normal (asosiy traffic)
        run: |
          aws s3 sync dist/ s3://app-prod/normal/ --delete
          aws cloudfront create-invalidation --distribution-id ABC --paths "/*"
      
      - name: Deploy profiling (1% sampling)
        run: |
          aws s3 sync dist-profiling/ s3://app-prod/profiling/ --delete
          aws cloudfront create-invalidation --distribution-id DEF --paths "/*"
```

CDN routing — sampling logic:

```nginx
# nginx — 1% traffic to profiling
http {
  split_clients "${remote_addr}${http_user_agent}" $variant {
    1%   profiling;
    *    normal;
  }
  
  server {
    location / {
      if ($variant = "profiling") {
        proxy_pass https://app-prod-profiling.cdn.com;
        break;
      }
      proxy_pass https://app-prod-normal.cdn.com;
    }
  }
}
```

Profiling build user'lardan data yig'ish:

```tsx
import { Profiler } from 'react';

const isProfilingBuild = (window as any).__PROFILING__;

const onRender = (id, phase, actualDuration, baseDuration) => {
  if (!isProfilingBuild) return;
  
  navigator.sendBeacon(
    '/api/monitoring/react',
    JSON.stringify({ id, phase, actualDuration, baseDuration })
  );
};

function App(): ReactElement {
  // Faqat profiling build'da Profiler wrap
  const content = <MainApp />;
  
  if (isProfilingBuild) {
    return <Profiler id="App" onRender={onRender}>{content}</Profiler>;
  }
  return content;
}
```

</details>

### Mashq 4: Web Vitals + React Profiler Correlation (O'rta)

INP > 200ms holatida, INP davomida render bo'lgan React komponent'larni topib log qilish. Slow INP root cause analysis pattern.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { Profiler } from 'react';
import { onINP } from 'web-vitals';
import type { ReactElement, ReactNode, ProfilerOnRenderCallback } from 'react';

interface RenderEvent {
  id: string;
  duration: number;
  timestamp: number;
}

const renderHistory: RenderEvent[] = [];
const HISTORY_LIMIT = 1000;

const onRender: ProfilerOnRenderCallback = (id, phase, actualDuration) => {
  renderHistory.push({
    id,
    duration: actualDuration,
    timestamp: performance.now(),
  });
  
  // Cap memory
  if (renderHistory.length > HISTORY_LIMIT) {
    renderHistory.shift();
  }
};

function setupINPCorrelation(): void {
  onINP((metric) => {
    if (metric.value <= 200) return; // Within budget
    
    const interactionEntries = metric.entries;
    if (interactionEntries.length === 0) return;
    
    const interactionStart = interactionEntries[0].startTime;
    const interactionEnd = interactionStart + metric.value;
    
    // Find React renders during this interaction
    const relatedRenders = renderHistory.filter(
      (r) => r.timestamp >= interactionStart && r.timestamp <= interactionEnd
    );
    
    // Sort by duration descending
    relatedRenders.sort((a, b) => b.duration - a.duration);
    
    const slowRenders = relatedRenders.filter((r) => r.duration > 16);
    
    if (slowRenders.length > 0) {
      console.warn(`[Slow INP ${metric.value}ms] React renders:`);
      slowRenders.forEach((r) => {
        console.warn(`  ${r.id}: ${r.duration.toFixed(2)}ms`);
      });
      
      // Send to monitoring
      navigator.sendBeacon(
        '/api/monitoring/slow-inp',
        JSON.stringify({
          inp: metric.value,
          interactionStart,
          interactionEnd,
          renders: slowRenders,
          totalRenderTime: slowRenders.reduce((s, r) => s + r.duration, 0),
        })
      );
    }
  });
}

interface MonitoredAppProps {
  children: ReactNode;
}

function MonitoredApp({ children }: MonitoredAppProps): ReactElement {
  // Setup once
  if (typeof window !== 'undefined' && !(window as any).__INP_INIT__) {
    setupINPCorrelation();
    (window as any).__INP_INIT__ = true;
  }
  
  return (
    <Profiler id="App" onRender={onRender}>
      {children}
    </Profiler>
  );
}

// Real-world test:
// User clicks "Submit" button
// Heavy validation + API call
// INP = 350ms (over budget)
//
// Console:
// [Slow INP 350ms] React renders:
//   ValidationPanel: 120ms
//   FormSummary: 80ms
//   App: 50ms
//
// Monitoring data:
// {
//   inp: 350,
//   renders: [
//     { id: 'ValidationPanel', duration: 120 },
//     { id: 'FormSummary', duration: 80 },
//     ...
//   ],
//   totalRenderTime: 250
// }
//
// Conclusion: 250ms of 350ms INP is React render time
// Optimization target: ValidationPanel (120ms)
```

</details>

### Mashq 5: Real-World Bottleneck Analysis (Qiyin)

`<SearchPage>` 1000+ items. User typing'ga lag bor. To'liq workflow: profile → analyze → optimize → re-profile → confirm. Bir nechta optimization apply qilish.

```tsx
import { useState } from 'react';
import type { ReactElement } from 'react';

interface Article {
  id: string;
  title: string;
  author: string;
  publishedAt: string;
  views: number;
}

function ArticleRow({
  article,
  onSelect,
  isHighlighted,
}: {
  article: Article;
  onSelect: (id: string) => void;
  isHighlighted: boolean;
}): ReactElement {
  return (
    <li
      onClick={() => onSelect(article.id)}
      style={{ background: isHighlighted ? '#ffe' : 'white' }}
    >
      <h4>{article.title}</h4>
      <small>{article.author} — {article.publishedAt}</small>
      <span>{article.views} views</span>
    </li>
  );
}

function SearchPage({ articles }: { articles: Article[] }): ReactElement {
  const [query, setQuery] = useState('');
  const [selectedId, setSelectedId] = useState<string | null>(null);
  const [sortBy, setSortBy] = useState<'date' | 'views' | 'title'>('date');
  
  const filtered = articles
    .filter((a) => a.title.toLowerCase().includes(query.toLowerCase()))
    .sort((a, b) => {
      if (sortBy === 'date') return b.publishedAt.localeCompare(a.publishedAt);
      if (sortBy === 'views') return b.views - a.views;
      return a.title.localeCompare(b.title);
    });
  
  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search articles..."
      />
      <select value={sortBy} onChange={(e) => setSortBy(e.target.value as typeof sortBy)}>
        <option value="date">Date</option>
        <option value="views">Views</option>
        <option value="title">Title</option>
      </select>
      <p>{filtered.length} articles</p>
      <ul>
        {filtered.map((a) => (
          <ArticleRow
            key={a.id}
            article={a}
            onSelect={setSelectedId}
            isHighlighted={a.id === selectedId}
          />
        ))}
      </ul>
    </div>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

### Step 1: Profile (initial)

DevTools Profiler:

- Type "react" character by character.
- Stop record.

Flame chart shows:
- SearchPage: ~80ms per keystroke
- ArticleRow × 1000: each ~0.05ms (total 50ms)

"Why did this render?":
- SearchPage: Hooks changed (query)
- ArticleRow: Parent component rendered

### Step 2: Analyze

Bottleneck:

1. Filter + sort her keystroke (`articles.filter().sort()`).
2. ArticleRow 1000 ta — top-down propagation, no memo.
3. `onSelect={setSelectedId}` stable (setState setter), lekin `isHighlighted` har gal hisoblanadi.

### Step 3: Apply Optimizations

```tsx
import { useState, useMemo, useCallback, memo, useTransition } from 'react';
import type { ReactElement } from 'react';

interface Article {
  id: string;
  title: string;
  author: string;
  publishedAt: string;
  views: number;
}

// 1. ArticleRow memo
const ArticleRow = memo(function ArticleRow({
  article,
  onSelect,
  isHighlighted,
}: {
  article: Article;
  onSelect: (id: string) => void;
  isHighlighted: boolean;
}): ReactElement {
  return (
    <li
      onClick={() => onSelect(article.id)}
      style={{ background: isHighlighted ? '#ffe' : 'white' }}
    >
      <h4>{article.title}</h4>
      <small>{article.author} — {article.publishedAt}</small>
      <span>{article.views} views</span>
    </li>
  );
});

function SearchPage({ articles }: { articles: Article[] }): ReactElement {
  const [query, setQuery] = useState('');
  const [deferredQuery, setDeferredQuery] = useState('');
  const [selectedId, setSelectedId] = useState<string | null>(null);
  const [sortBy, setSortBy] = useState<'date' | 'views' | 'title'>('date');
  const [isPending, startTransition] = useTransition();
  
  // 2. useTransition — query non-urgent
  function handleQueryChange(value: string) {
    setQuery(value);
    startTransition(() => {
      setDeferredQuery(value);
    });
  }
  
  // 3. useMemo — filter + sort
  const filtered = useMemo(() => {
    return articles
      .filter((a) => a.title.toLowerCase().includes(deferredQuery.toLowerCase()))
      .sort((a, b) => {
        if (sortBy === 'date') return b.publishedAt.localeCompare(a.publishedAt);
        if (sortBy === 'views') return b.views - a.views;
        return a.title.localeCompare(b.title);
      });
  }, [articles, deferredQuery, sortBy]);
  
  // 4. useCallback — onSelect (already stable, but explicit for clarity)
  // setState setter stable forever — no useCallback needed
  
  return (
    <div>
      <input
        value={query}
        onChange={(e) => handleQueryChange(e.target.value)}
        placeholder="Search articles..."
      />
      <select value={sortBy} onChange={(e) => setSortBy(e.target.value as typeof sortBy)}>
        <option value="date">Date</option>
        <option value="views">Views</option>
        <option value="title">Title</option>
      </select>
      <p>{filtered.length} articles {isPending && '(filtering...)'}</p>
      <ul style={{ opacity: isPending ? 0.5 : 1 }}>
        {filtered.map((a) => (
          <ArticleRow
            key={a.id}
            article={a}
            onSelect={setSelectedId}
            isHighlighted={a.id === selectedId}
          />
        ))}
      </ul>
    </div>
  );
}

// 5. Future: virtualization (react-window) for 10000+ items
import { FixedSizeList } from 'react-window';

function SearchPageVirtualized({ articles }: { articles: Article[] }): ReactElement {
  // ... state
  
  function Row({ index, style }: { index: number; style: React.CSSProperties }): ReactElement {
    const article = filtered[index];
    return (
      <div style={style}>
        <ArticleRow
          article={article}
          onSelect={setSelectedId}
          isHighlighted={article.id === selectedId}
        />
      </div>
    );
  }
  
  return (
    <div>
      {/* ... input + select */}
      <FixedSizeList
        height={600}
        itemCount={filtered.length}
        itemSize={80}
        width="100%"
      >
        {Row}
      </FixedSizeList>
    </div>
  );
}
```

### Step 4: Re-Profile

After optimizations:

- Type "react":
  - SearchPage: ~2ms (input update only)
  - ArticleRow re-render: faqat selectedId match'ga tegishli row
  - Filter: deferred (Transition Lane)

Flame chart:
- SearchPage: 2ms
- ArticleRow (selected): 0.5ms
- Other ArticleRows: bailout (memo)

Improvement:
- Before: ~80ms per keystroke
- After: ~2ms input + ~10-30ms filter (Transition Lane, interruptible)
- INP: < 100ms (target met)

### Step 5: Production Monitoring

```tsx
import { Profiler } from 'react';

const onRender = (id, phase, actualDuration) => {
  if (actualDuration > 50) {
    monitoring.send({ type: 'slow-react', id, duration: actualDuration });
  }
};

<Profiler id="SearchPage" onRender={onRender}>
  <SearchPage articles={articles} />
</Profiler>
```

React Compiler era — soddalashtirish:

```tsx
'use memo';

// Compiler avtomatik:
// - filtered cache
// - ArticleRow memo equivalent
// - selectedId comparison

function SearchPage({ articles }: { articles: Article[] }): ReactElement {
  const [query, setQuery] = useState('');
  const [deferredQuery, setDeferredQuery] = useState('');
  const [selectedId, setSelectedId] = useState<string | null>(null);
  const [sortBy, setSortBy] = useState<'date' | 'views' | 'title'>('date');
  const [isPending, startTransition] = useTransition();
  
  function handleQueryChange(value: string) {
    setQuery(value);
    startTransition(() => setDeferredQuery(value));
  }
  
  const filtered = articles.filter(/* ... */).sort(/* ... */);
  
  return /* ... */;
}
```

Compiler manual `useMemo`/`memo`/`useCallback`'ni avtomatik qiladi.

</details>

---

## Xulosa

Bu fayl React profiling'ning to'liq spectrum'ini o'rgandi:

- **Profiling Concept** — measurement-based optimization, hot path identification, real-world impact, regression detection.
- **React DevTools Profiler** — installation (browser extension), Components vs Profiler tab, settings yoqish (Record why each component rendered, Highlight Updates).
- **Recording Workflow** — symptom → reproduce → record → analyze → re-profile → confirm → production monitoring.
- **Flame Chart** — top-down structural view, width = duration, color **commit'ga nisbatan** (cool → hot, eng sekin fiber rang scale anchor), self vs tree time.
- **Ranked Chart** — slowest first, optimization priority.
- **Commit Details** — props/state/hooks inspection, source location, diff view.
- **"Why did this render?"** — re-render reasons (first time, hooks changed, props changed, parent rendered, context changed). Optimization strategy jadval.
- **`<Profiler>` Component API** — programmatic profiling, nested Profilers, production telemetry. Default no-op in production, `react-dom/profiling` build'da enabled.
- **onRender Callback Fields** — id, phase (mount/update/nested-update), actualDuration, baseDuration, startTime, commitTime. Memoization efficiency formula `(baseDuration - actualDuration) / baseDuration`.
- **Production Profiling** — `react-dom/profiling` Vite/Webpack/Next.js setup, sampling deployment (1% users).
- **Web Vitals Integration** — Core Web Vitals (LCP, INP, CLS), 2024-yil mart oyida FID o'rniga INP qo'shildi (FID → INP transition March 2024 — Google Web Vitals, React'dan mustaqil), `web-vitals` package, INP correlation with React renders.
- **Real-World Workflow** — 8-step (symptom → reproduce → local repro → profile → analyze → hypothesis+optimize → re-profile → deploy+monitor).
- **Highlight Updates** — visual debug, rang **frequency-based** (cool yashil — kam-uchraydigan / sariq — moderate / qizil — hot spot, tez-tez re-render). Production'da yo'q. Strict Mode 2x bitta commit ichida ikki function call (alohida highlight signal emas).
- **Strict Mode Profile Impact** — development 2x render, profile data ~2x kattaroq. Production build profile yoki conditional StrictMode.
- **Performance Budget** — React-specific (component render p99, total commit p95), Web Vitals (LCP/INP/CLS), bundle size. Real-time + aggregated alerting, CI/CD integration (Lighthouse CI).

QISM 8 (Performance & Compiler) — 4/6 yozildi. Keyingi fayl `35-code-splitting.md` — `React.lazy` chuqur, route-based splitting, prefetching strategies (hover/idle/viewport), R19 Preloading APIs (`preload`, `preinit`, `prefetchDNS`, `preconnect`).

---

**Keyingi bo'lim:** [35-code-splitting.md](35-code-splitting.md) — Code splitting strategies. `React.lazy` deep dive (`29-suspense-lazy.md` base'dan), route-based splitting (React Router lazy loading), feature-based splitting (HeavyEditor, Modal, Chart on demand), vendor splitting (webpack splitChunks, Vite manualChunks), magic comments (webpackChunkName, webpackPrefetch, webpackPreload), R19 Document API preload (`preload`, `preinit`, `prefetchDNS`, `preconnect`) programmatic, hover/focus/idle preloading patterns (requestIdleCallback), service worker chunk caching, bundle analyzer workflow.
