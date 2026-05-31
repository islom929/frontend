# Bo'lim 5: Scheduler & Lanes

> Scheduler — Reconciler render ishini **priority bilan boshqaruvchi** mexanizm. Lanes — R18+ priority systemsi: 31-bit bitmap orqali bir vaqtda bir nechta priority'ni saqlash. Bu bo'lim concurrent rendering'ning asosini yoritadi: priority levels, time slicing, interruptible rendering, expiration (starvation prevention), MessageChannel infrastructure.

---

## Mundarija

- [Concurrent Rendering Nima Uchun Kerak](#concurrent-rendering-nima-uchun-kerak)
- [Scheduler Package — Alohida `scheduler/`](#scheduler-package--alohida-scheduler)
- [Priority Levels](#priority-levels)
- [Lanes Model (R18+)](#lanes-model-r18)
- [Lane Types](#lane-types)
- [Time Slicing va Frame Budget](#time-slicing-va-frame-budget)
- [Interruptible Rendering](#interruptible-rendering)
- [Expiration va Starvation Prevention](#expiration-va-starvation-prevention)
- [MessageChannel vs requestIdleCallback](#messagechannel-vs-requestidlecallback)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Concurrent Rendering Nima Uchun Kerak

### Nazariya

JavaScript **single-threaded** — bir vaqtda bitta ish bajariladi. Browserda asosiy thread (main thread) quyidagi ishlarni navbat bilan bajaradi:

1. **JavaScript ijrosi** — sizning kodingiz, React render
2. **Style recalculation** — CSS qoidalari hisob-kitobi
3. **Layout** — har element pozitsiyasi va o'lchami
4. **Paint** — pixel'lar ekranga
5. **Composite** — layer'lar birlashtiruvi

Browser bu ishlarni **60 fps** (har 16.67ms da bitta frame) tezligida bajarishi kerak — silliq UI uchun. Agar JavaScript 16ms'dan ko'proq vaqt olsa, browser frame'ni o'tkazib yuboradi (jank).

**Pre-Concurrent muammosi:**

```tsx
function HeavyList({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map(item => <ComplexCard key={item.id} item={item} />)}
    </ul>
  );
}

// 10,000 ta item, har biri ~0.1ms render → jami ~1000ms
// Sync render bo'lsa:
// 1. setState chaqiriladi
// 2. React 1000ms davomida render qiladi
// 3. Bu 1000ms davomida JavaScript thread BLOKLANADI:
//    - Input javob bermaydi (click, scroll, type)
//    - Animation muzlaydi (requestAnimationFrame chaqirilmaydi)
//    - Paint to'xtaydi (yangi frame yo'q)
// 4. 1000ms (~60 frame!) o'tkazib yuborildi
// 5. Foydalanuvchi UI'ni "muzlagan" deb sezadi
```

**Concurrent Rendering yechimi:**

React render'ni **bo'lib-bo'lib** bajaradi:

```
Concurrent rendering:
1. setState → render rejalashtirildi
2. Render boshlanadi (5ms ishladi)
3. shouldYield() → browser'ga yo'l berish
4. Browser ishi (input, paint) bajariladi
5. MessageChannel orqali React qayta uyg'onadi
6. Render davom etadi (5ms ishladi)
7. Yana yield...
8. Bu cycle render tugamaguncha takrorlanadi
9. Render tugagach — atomic commit
```

**Foydalari:**

1. **UI doim javob beradi** — input, scroll, click har 5ms da javob oladi
2. **Animation silliq** — frame'lar o'tkazib yuborilmaydi (60 fps saqlanadi)
3. **Priority bilan ish** — urgent update'lar (typing) low-priority update'lardan oldin
4. **Tashlanuvchi render** — yangi update kelganda eski "yo'lda turgan" render tashlanadi

**Cheklovlar:**

1. **Render pure bo'lishi shart** — uziluvchi va qayta boshlanuvchi (cross-ref `02-rendering.md` Render Phase)
2. **Effect'lar idempotent bo'lishi kerak** — Strict Mode 2x effect cycle bilan tekshiriladi
3. **External store subscription** `useSyncExternalStore` orqali (tearing prevention)

> **Eslatma:** Concurrent rendering — **opt-in** asosida. `createRoot` (R18+) ishlatilsa avtomatik yoqiladi. R18'da `ReactDOM.render` (legacy) ishlatilsa — concurrent yo'q (sync mode). **R19'da `ReactDOM.render`, `ReactDOM.hydrate` va `ReactDOM.unmountComponentAtNode` butunlay olib tashlandi** — `createRoot` / `hydrateRoot` majburiy.

<details>
<summary><strong>Under the Hood</strong></summary>

**Browser event loop va React Scheduler:**

```
Browser event loop (60 fps target):
┌─────────────────────────────────────────┐
│  Macrotask (script, setTimeout, ...)   │  ← React work shu yerda
├─────────────────────────────────────────┤
│  Microtask queue flush                  │
├─────────────────────────────────────────┤
│  Animation frame callbacks (rAF)        │
├─────────────────────────────────────────┤
│  Style recalculation                    │
├─────────────────────────────────────────┤
│  Layout                                 │
├─────────────────────────────────────────┤
│  Paint                                  │
├─────────────────────────────────────────┤
│  Composite                              │
└─────────────────────────────────────────┘
                    ↓
            Next frame (16.67ms later)
```

React Scheduler **macrotask boshida** ishlaydi. Har 5ms ish qilgandan keyin:
1. `shouldYield()` true qaytaradi
2. React work loop'ni to'xtatadi
3. `MessageChannel.port2.postMessage(null)` chaqiriladi
4. Joriy macrotask tugaydi
5. Microtask queue ishlanadi
6. Browser rendering pipeline (rAF, style, layout, paint)
7. Yangi macrotask boshida `port1.onmessage` chaqiriladi → React davom etadi

**`shouldYield()` implementation:**

```typescript
// scheduler internal (soddalashtirilgan)
let frameStartTime = -1;
const frameYieldMs = 5;

function shouldYieldToHost(): boolean {
  const timeElapsed = getCurrentTime() - frameStartTime;
  if (timeElapsed < frameYieldMs) {
    return false;  // Hali budget bor
  }
  
  // Budget tugadi — yield kerak
  return true;
}
```

**`scheduler` paket:**

React `scheduler` deb nomlangan **alohida npm paket**ga bog'liq. Bu paket:
- Priority-based callback scheduling
- Time slicing
- MessageChannel-based yielding

Public API:

```typescript
import {
  unstable_scheduleCallback,
  unstable_cancelCallback,
  unstable_runWithPriority,
  unstable_NormalPriority,
  unstable_UserBlockingPriority,
  // ...
} from 'scheduler';

const handle = unstable_scheduleCallback(
  unstable_NormalPriority,
  () => { /* work */ },
);

// Cancel
unstable_cancelCallback(handle);
```

`unstable_` prefix'i — "API barqaror emas, faqat React internal'lar uchun mo'ljallangan" degani. App kodda ishlatish noto'g'ri.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Concurrent rendering ko'rgazma — useTransition bilan:

```tsx
import { useState, useTransition } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<Item[]>([]);
  const [isPending, startTransition] = useTransition();

  function handleChange(value: string) {
    // URGENT update — input darhol javob beradi
    setQuery(value);
    
    // NON-URGENT update — render uziluvchi, kechiktirish mumkin
    startTransition(() => {
      const filtered = expensiveFilter(allItems, value);
      setResults(filtered);
    });
  }

  return (
    <>
      <input
        value={query}
        onChange={e => handleChange(e.target.value)}
        placeholder="Qidirish..."
      />
      {isPending && <Spinner />}
      <ResultList items={results} />
    </>
  );
}

// Foydalanuvchi tezda yozadi: "a", "ab", "abc"
//
// Sync rendering bo'lganda:
// 1. "a" → 200ms render → input bloklanadi
// 2. "ab" → 200ms render → input bloklanadi
// 3. "abc" → 200ms render → input bloklanadi
// → Foydalanuvchi UI'ni "muzlagan" deb sezadi
//
// Concurrent rendering + useTransition:
// 1. "a" → setQuery (urgent, input darhol updated)
//    → startTransition: results render boshlanadi (low-priority)
// 2. "ab" → setQuery (urgent)
//    → "a" results render TASHLANADI (yangi update keldi)
//    → "ab" results render boshlanadi
// 3. "abc" → setQuery (urgent)
//    → "ab" render tashlanadi → "abc" boshlanadi
// 4. Foydalanuvchi to'xtagach — render tugaydi
// → Input doim darhol javob beradi
```

Pre-concurrent vs concurrent log misol:

```tsx
function App() {
  const [filter, setFilter] = useState('');
  const [items] = useState(() => generateItems(10000));

  // Sync rendering
  function handleChangeSync(e: React.ChangeEvent<HTMLInputElement>) {
    console.time('render');
    setFilter(e.target.value);  // 10,000 ta item filter va render
    console.timeEnd('render');
  }

  // Concurrent rendering
  const [, startTransition] = useTransition();
  function handleChangeConcurrent(e: React.ChangeEvent<HTMLInputElement>) {
    startTransition(() => {
      console.time('render-transition');
      setFilter(e.target.value);
      console.timeEnd('render-transition');
    });
  }

  const filtered = items.filter(i => i.name.includes(filter));

  return (
    <>
      <input onChange={handleChangeConcurrent} />
      <ul>
        {filtered.map(item => <ComplexCard key={item.id} item={item} />)}
      </ul>
    </>
  );
}

// Sync: "render: 800ms" — har keypress'da
// Transition: "render-transition: 5ms, 5ms, 5ms, ..." — chunk'lar bilan, jami ~800ms lekin uziluvchi
```

</details>

---

## Scheduler Package — Alohida `scheduler/`

### Nazariya

`scheduler` — React'ning **alohida npm paketi**. U React Reconciler'dan mustaqil ishlash uchun mo'ljallangan: priority-based task scheduling va time slicing.

**Paket tuzilishi:**

```
scheduler/
├── unstable_scheduleCallback        // Task rejalashtirish
├── unstable_cancelCallback          // Task bekor qilish
├── unstable_runWithPriority         // Wrapper bilan priority
├── unstable_getCurrentPriorityLevel // Joriy priority
├── unstable_now                     // performance.now() polyfill
├── unstable_shouldYield             // Yield kerakmi
├── unstable_continueExecution       // Davom ettirish
├── unstable_pauseExecution          // To'xtatish
├── unstable_ImmediatePriority       // = 1
├── unstable_UserBlockingPriority    // = 2
├── unstable_NormalPriority          // = 3
├── unstable_LowPriority             // = 4
└── unstable_IdlePriority            // = 5
```

**Nima uchun alohida paket:**

1. **Separation of concerns** — scheduler React'ga maxsus emas. Boshqa library'lar ham ishlatishi mumkin (Reach UI, va h.k.)
2. **Independent versioning** — scheduler R16+ uchun bir xil. React versiya yangilansa, scheduler stable qoladi
3. **Test isolation** — scheduler unit test'lar React'siz yoziladi
4. **Custom renderer'lar** — Ink, react-three-fiber kabi renderer'lar ham scheduler ishlatadi

**Public API — `unstable_*` prefix'i:**

`unstable_` prefix'i React jamoasining **"barqaror emas"** signali. Bu API'lar:
- React internal'lar uchun mo'ljallangan
- App kod ishlatmasligi kerak
- Versiyalar orasida o'zgarishi mumkin (breaking changes warning'siz)

App developer'lar `scheduler`'ni to'g'ridan-to'g'ri ishlatmaydi — React `useTransition`, `startTransition` orqali wrapping qiladi.

### Scheduler ishlash printsipi

**Asosiy mexanizm:**

1. **Task qatori (priority queue)** — har task'ning priority darajasi bor
2. **Sortlash** — high-priority task'lar oldinda
3. **Yield-friendly execution** — har 5ms'dan keyin browser'ga yo'l berish
4. **Cancellation** — task bekor qilish mumkin

```typescript
// scheduler API ko'rgazma
import {
  unstable_scheduleCallback as scheduleCallback,
  unstable_NormalPriority as NormalPriority,
  unstable_UserBlockingPriority as UserBlockingPriority,
} from 'scheduler';

// Normal priority task
const task1 = scheduleCallback(NormalPriority, () => {
  console.log('Normal task');
});

// User blocking (yuqori priority)
const task2 = scheduleCallback(UserBlockingPriority, () => {
  console.log('Urgent task');
});

// Callback continuation function qaytarsa, scheduler uni qayta chaqiradi:
function processQueue(didTimeout) {
  // Heavy work — qism-qism bajarish
  while (workQueue.length > 0 && !shouldYield()) {
    processItem(workQueue.shift());
  }

  if (workQueue.length > 0) {
    // Hali ish bor — continuation FUNCTION qaytariladi (task handle emas).
    // workLoop `typeof result === 'function'` tekshiradi va shu funksiyani
    // joriy task'ning callback'i sifatida qayta ishlatadi
    return processQueue;
  }

  // Tugatdi — null qaytarsa task qatordan olib tashlanadi
  return null;
}

const task3 = scheduleCallback(NormalPriority, processQueue);
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Scheduler internal structure:**

```typescript
// scheduler internal (soddalashtirilgan)

// Task interface
interface Task {
  id: number;
  callback: (didTimeout: boolean) => void | (() => void);
  priorityLevel: number;
  startTime: number;
  expirationTime: number;
  sortIndex: number;  // Min-heap sort
}

// Ikki task qator
const taskQueue: Task[] = [];     // Tayyor task'lar (min-heap by sortIndex)
const timerQueue: Task[] = [];    // Kechiktirilgan task'lar (delay bilan)

let isPerformingWork = false;
let isHostCallbackScheduled = false;

function scheduleCallback(priorityLevel, callback, options) {
  const currentTime = getCurrentTime();
  
  // Priority'ga qarab timeout
  let timeout;
  switch (priorityLevel) {
    case ImmediatePriority:    timeout = -1; break;        // Darhol
    case UserBlockingPriority: timeout = 250; break;       // 250ms
    case NormalPriority:       timeout = 5000; break;      // 5s
    case LowPriority:          timeout = 10000; break;     // 10s
    case IdlePriority:         timeout = 1073741823; break; // maxSigned31BitInt (2^30 - 1) ≈ 12.4 kun (effectively never)
  }
  
  const expirationTime = currentTime + timeout;
  
  const newTask = {
    id: taskIdCounter++,
    callback,
    priorityLevel,
    startTime: currentTime,
    expirationTime,
    sortIndex: expirationTime,  // Min-heap orderly
  };
  
  push(taskQueue, newTask);
  
  if (!isHostCallbackScheduled && !isPerformingWork) {
    isHostCallbackScheduled = true;
    requestHostCallback();
  }
  
  return newTask;
}
```

**Min-heap priority queue:**

`taskQueue` — **min-heap** structure. Eng past `sortIndex` (ya'ni eng yaqin `expirationTime`) eng oldinda. Push/pop O(log n).

Bu — performance optimizationsi: har scheduling'da heap'ga insert O(log n), eng yuqori priority task'ni olish O(log n).

**Work loop:**

```typescript
function workLoop(initialTime) {
  let currentTime = initialTime;
  let currentTask = peek(taskQueue);  // Eng yuqori priority
  
  while (currentTask !== null) {
    if (currentTask.expirationTime > currentTime && shouldYieldToHost()) {
      // Hali expire bo'lmagan va budget tugagan — yield
      break;
    }
    
    const callback = currentTask.callback;
    if (typeof callback === 'function') {
      currentTask.callback = null;
      currentPriorityLevel = currentTask.priorityLevel;
      
      const didTimeout = currentTask.expirationTime <= currentTime;
      const continuationCallback = callback(didTimeout);
      
      if (typeof continuationCallback === 'function') {
        // Task continuation qaytardi — yana qatorga qo'shish
        currentTask.callback = continuationCallback;
      } else {
        if (currentTask === peek(taskQueue)) {
          pop(taskQueue);
        }
      }
    } else {
      pop(taskQueue);
    }
    
    currentTask = peek(taskQueue);
  }
  
  // Hali task bo'lsa — keyingi tick uchun rejalashtirish
  if (currentTask !== null) return true;
  return false;
}
```

**`requestHostCallback` va MessageChannel:**

```typescript
let scheduledHostCallback = null;

function requestHostCallback() {
  scheduledHostCallback = workLoop;
  schedulePerformWorkUntilDeadline();
}

const channel = new MessageChannel();
const port = channel.port2;

channel.port1.onmessage = function performWorkUntilDeadline() {
  if (scheduledHostCallback !== null) {
    const currentTime = getCurrentTime();
    
    // Frame budget'ni o'rnatish
    frameStartTime = currentTime;
    deadline = currentTime + frameYieldMs;
    
    const hasMoreWork = scheduledHostCallback(currentTime);
    
    if (hasMoreWork) {
      // Yana ish bor — keyingi tick uchun rejalashtirish
      port.postMessage(null);
    } else {
      isMessageLoopRunning = false;
      scheduledHostCallback = null;
    }
  }
};

function schedulePerformWorkUntilDeadline() {
  port.postMessage(null);
}
```

`MessageChannel.postMessage()` — yangi macrotask rejalashtiradi. Browser bu macrotask'ni rendering pipeline'dan **keyin** ishlaydi (paint bo'lgandan keyin).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Scheduler ishlash misol (low-level, faqat ko'rgazma uchun):

```typescript
// ⚠️ Bu kod — faqat ko'rgazma. App kodda ishlatish tavsiya qilinmaydi
import {
  unstable_scheduleCallback as scheduleCallback,
  unstable_NormalPriority as NormalPriority,
  unstable_UserBlockingPriority as UserBlockingPriority,
  unstable_shouldYield as shouldYield,
} from 'scheduler';

const items = generateLargeArray(1_000_000);
const results: number[] = [];
let index = 0;

function processChunk(): null | (() => null | typeof processChunk) {
  while (index < items.length && !shouldYield()) {
    results.push(items[index] * 2);
    index++;
  }
  
  if (index < items.length) {
    // Hali ish bor — continuation
    return processChunk;  // Qaytarib chaqiriladi
  }
  
  // Tugatdi
  console.log('Done', results.length);
  return null;
}

scheduleCallback(NormalPriority, processChunk);
// 1M item ishlanadi qism-qism, browser bloklanmasdan
```

React `useTransition` ichida shu mexanizm ishlatiladi — `unstable_scheduleCallback` to'g'ridan-to'g'ri yozilmaydi, balki React API'lari orqali ishlatiladi:

```tsx
// React level — app kodi
function App() {
  const [, startTransition] = useTransition();
  
  function handleChange(value: string) {
    startTransition(() => {
      // Bu callback ichidagi setState'lar — TransitionLane priority
      // Scheduler tomonidan low-priority task'larga rejalashtiriladi
      setHeavyState(value);
    });
  }
  
  return <input onChange={e => handleChange(e.target.value)} />;
}
```

Internal'da `startTransition`:

```typescript
// React internal (soddalashtirilgan)
function startTransition(scope) {
  const prevTransition = ReactCurrentBatchConfig.transition;
  ReactCurrentBatchConfig.transition = {};  // Transition flag o'rnatish
  
  try {
    scope();  // setState'lar TransitionLane bilan rejalashtiriladi
  } finally {
    ReactCurrentBatchConfig.transition = prevTransition;
  }
}
```

</details>

---

## Priority Levels

### Nazariya

Scheduler **5 ta priority level**'ni qo'llab-quvvatlaydi:

| Priority | Constant | Timeout | Vazifa |
|----------|----------|---------|--------|
| **1. Immediate** | `unstable_ImmediatePriority` | -1 (darhol) | Sync, yield qilmaydi (synchronous urgency) |
| **2. UserBlocking** | `unstable_UserBlockingPriority` | 250ms | Foydalanuvchi javob kutadi (click, type, hover) |
| **3. Normal** | `unstable_NormalPriority` | 5000ms | Default — odatdagi update'lar (data fetch result) |
| **4. Low** | `unstable_LowPriority` | 10000ms | Kechiktirish mumkin (background data) |
| **5. Idle** | `unstable_IdlePriority` | `maxSigned31BitInt` ≈ 12.4 kun (effectively hech qachon) | Faqat hech narsa bo'lmaganda (telemetry) |

**Timeout ma'nosi:**

Timeout — task'ning **expirationTime** ni hisoblash uchun ishlatiladi. `expirationTime = currentTime + timeout`. Bu — "task qachon expire bo'ladi" degan vaqt. Expire bo'lgandan keyin task **higher priority** sifatida ishlanadi (starvation prevention).

Misol:
- Normal priority task — 5 sekundgacha kutilishi mumkin
- 5 sekund o'tgach — expired deb belgilanadi
- Endi sync tarzda bajariladi (yield qilmasdan)

**Priority boshlanishi:**

Har scheduling'da **default — Normal priority**. Lekin React boshqa priority'lardan ham foydalanadi:

- **Click handler ichidagi setState** → `SyncLane` (Immediate priority bilan birga)
- **`useTransition` ichidagi setState** → `TransitionLane` (Normal priority bilan birga)
- **Scroll handler ichidagi setState** → `InputContinuousLane` (UserBlocking priority bilan birga)
- **`requestIdleCallback`-ga o'xshash** → `IdleLane`

### Priority darajasi va Lane — IKKI ALOHIDA SYSTEM

React'da priority'ning **ikki alohida systemsi** mavjud va ular tez-tez chalkashtiriladi:

1. **Scheduler priority** — `scheduler` paketi darajasida (`ImmediatePriority` … `IdlePriority`). Task'ni qaysi tezlikda macrotask qilib chaqirishni boshqaradi.
2. **React Lanes** — `react-reconciler` darajasida (`SyncLane`, `DefaultLane`, `TransitionLane`, …). Fiber tree ichida har update'ning priority'sini bitmask sifatida saqlaydi.

R18'da Lane systemsi kiritildi va Reconciler ichki priority sifatida Lane'larni ishlatadi. Lekin Scheduler priority hali ham mavjud — Reconciler `scheduleCallback` chaqirganda Lane'ni Scheduler priority'ga **map** qiladi (`lanesToEventPriority` va teskari yo'l `eventPriorityToLane`):

```typescript
// Lane → Scheduler priority (mental model)
function lanesToSchedulerPriority(lanes) {
  if (lanes & (SyncLane | SyncHydrationLane)) return ImmediatePriority;
  if (lanes & (InputContinuousLane | InputContinuousHydrationLane)) return UserBlockingPriority;
  if (lanes & (DefaultLane | DefaultHydrationLane)) return NormalPriority;
  if (lanes & TransitionLanes) return NormalPriority;
  if (lanes & (IdleLane | IdleHydrationLane)) return IdlePriority;
  return NormalPriority;
}
```

Asosiy farqlar:
- **Lanes** — bitmask, bir Fiber'da bir vaqtda bir nechta priority
- **Scheduler priority** — har task uchun bittagina enum qiymat
- Lanes'da `TransitionLane1`…`TransitionLane16` farqli lane'lar — Scheduler'da hammasi `NormalPriority`

> **Eslatma:** "Sync render" so'zi ham chalkashlik chiqaradi: `SyncLane` (Reconciler lane) `ImmediatePriority` (Scheduler) bilan o'xshash, lekin teng emas. `SyncLane` — render Phase'da yield qilinmaydi, balki Scheduler ham `ImmediatePriority` task'larni microtask flush bilan birga ishlaydi (alohida macrotask kutmaydi).

<details>
<summary><strong>Under the Hood</strong></summary>

**Priority constant qiymatlari:**

```typescript
// scheduler/src/SchedulerPriorities.js
export const NoPriority = 0;
export const ImmediatePriority = 1;
export const UserBlockingPriority = 2;
export const NormalPriority = 3;
export const LowPriority = 4;
export const IdlePriority = 5;
```

**Timeout konstanta'lari:**

```typescript
// scheduler/src/forks/Scheduler.js
const IMMEDIATE_PRIORITY_TIMEOUT = -1;
const USER_BLOCKING_PRIORITY_TIMEOUT = 250;
const NORMAL_PRIORITY_TIMEOUT = 5000;
const LOW_PRIORITY_TIMEOUT = 10000;
const IDLE_PRIORITY_TIMEOUT = maxSigned31BitInt;  // = 1073741823 (2^30 - 1, ≈12.4 kun)
```

**`unstable_runWithPriority` ishlash:**

```typescript
let currentPriorityLevel = NormalPriority;

function runWithPriority(priorityLevel, eventHandler) {
  switch (priorityLevel) {
    case ImmediatePriority:
    case UserBlockingPriority:
    case NormalPriority:
    case LowPriority:
    case IdlePriority:
      break;
    default:
      priorityLevel = NormalPriority;
  }

  const previousPriorityLevel = currentPriorityLevel;
  currentPriorityLevel = priorityLevel;

  try {
    return eventHandler();
  } finally {
    currentPriorityLevel = previousPriorityLevel;
  }
}

// Misol — wrapping
runWithPriority(UserBlockingPriority, () => {
  // Bu callback ichidagi scheduleCallback'lar UserBlockingPriority bilan rejalashtiriladi
  scheduleCallback(getCurrentPriorityLevel(), work);
});
```

React internal'larda `runWithPriority` event handler'lar uchun ishlatilgan. R18+'da Lane systemsi bilan almashtirildi (ko'pchilik holatlarda), lekin scheduler API hali mavjud.

**Priority inheritance:**

`scheduleCallback` chaqirilganda, agar priority parametr berilmagan bo'lsa, `getCurrentPriorityLevel()` ishlatiladi (joriy context priority). Bu nested calls uchun foydali:

```typescript
runWithPriority(UserBlockingPriority, () => {
  // Joriy priority = UserBlocking
  scheduleCallback(getCurrentPriorityLevel(), () => {
    // Bu task UserBlocking sifatida rejalashtirildi
    runWithPriority(LowPriority, () => {
      // Joriy priority = Low (override)
      scheduleCallback(getCurrentPriorityLevel(), () => {
        // Bu task Low sifatida rejalashtirildi
      });
    });
  });
});
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Priority darajasi React API'lar orqali (app kodida):

```tsx
import { useState, useTransition, useDeferredValue, startTransition } from 'react';

function App() {
  const [count, setCount] = useState(0);
  const [results, setResults] = useState<Item[]>([]);
  const [isPending, startTrans] = useTransition();

  // 1. Click handler — SyncLane (Immediate priority)
  function handleClick() {
    setCount(c => c + 1);  // Sync, yield qilinmaydi
  }

  // 2. useTransition — TransitionLane (Normal priority)
  function handleSearch(query: string) {
    startTrans(() => {
      setResults(filter(query));  // Low-priority, uziluvchi
    });
  }

  // 3. Idle work — startTransition + custom logic
  function handleAnalytics() {
    startTransition(() => {
      // Background telemetry
      sendAnalytics(count);
    });
  }

  return (
    <>
      <button onClick={handleClick}>Click: {count}</button>
      <input onChange={e => handleSearch(e.target.value)} />
      {isPending && <p>Yangilanmoqda...</p>}
    </>
  );
}
```

`useDeferredValue` — implicit transition:

```tsx
function SearchPage() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  
  // deferredQuery — kechiktirilgan qiymat
  // Yangi query (urgent) → input darhol updated
  // deferredQuery (TransitionLane) → kelishi mumkin keyin
  
  const results = useMemo(
    () => expensiveSearch(deferredQuery),
    [deferredQuery]
  );
  
  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <ResultList items={results} />
    </>
  );
}
```

Real-world misol — autocomplete:

```tsx
function Autocomplete() {
  const [input, setInput] = useState('');
  const [suggestions, setSuggestions] = useState<string[]>([]);
  const [isPending, startTransition] = useTransition();

  function handleInput(value: string) {
    // Input value — urgent (foydalanuvchi yozayotganini darhol ko'radi)
    setInput(value);
    
    // Suggestions — non-urgent (1-2 frame kechikishi OK)
    startTransition(() => {
      const matched = ALL_OPTIONS.filter(opt =>
        opt.toLowerCase().includes(value.toLowerCase())
      );
      setSuggestions(matched);
    });
  }

  return (
    <>
      <input value={input} onChange={e => handleInput(e.target.value)} />
      {isPending && <Spinner />}
      <ul>
        {suggestions.map(s => <li key={s}>{s}</li>)}
      </ul>
    </>
  );
}

// Foydalanuvchi tezda yozayotganda:
// - Har keypress'da input darhol updated (SyncLane)
// - Suggestions update kechiktiriladi (TransitionLane)
// - Yangi keypress kelganda eski suggestions render TASHLANADI
// - Foydalanuvchi to'xtagach — suggestions ko'rsatildi
```

</details>

---

## Lanes Model (R18+)

### Nazariya

**Lanes** — R18'da kiritilgan yangi priority systemsi. Eski "expirationTime" model bitta number edi (har update uchun); Lanes — **31-bit bitmap**: har bit alohida lane (priority darajasi).

**Asosiy g'oya:**

```typescript
// Eski (R17): expirationTime
fiber.expirationTime = 1500;  // bitta number, bitta priority

// Yangi (R18+): lanes bitmap
fiber.lanes = SyncLane | TransitionLane;  // bir vaqtda 2 priority
```

**Bitmap nima uchun:**

1. **Bir vaqtda bir nechta priority** — bir Fiber'ga sync update va transition update kelishi mumkin
2. **Tezkor tekshiruv** — `lanes & SyncLane` bitwise AND O(1)
3. **Combined operations** — `lanes |= newLane` (qo'shish), `lanes &= ~lane` (olib tashlash)
4. **Priority hierarchy** — past bit → yuqori priority

### Lane bitmap structure

```
31-bit bitmap (eng past indexli bit = eng yuqori priority):
SyncHydrationLane            ← Eng yuqori priority
SyncLane
InputContinuousHydrationLane
InputContinuousLane
DefaultHydrationLane
DefaultLane
TransitionHydrationLane
TransitionLane1 .. TransitionLane16   (16 ta alohida lane)
RetryLane1 .. RetryLane4              (4 ta retry lane)
SelectiveHydrationLane
IdleHydrationLane
IdleLane
OffscreenLane                ← Eng past priority
```

> **Eslatma:** Aniq bit qiymatlari R18-R19 manba kodida (`react-reconciler/src/ReactFiberLane.js`). 31-bit cheklov — **V8 SMI (Small Integer) optimization** uchun: 31-bit signed integer V8'da pointer slot ichida tagged integer sifatida saqlanadi (heap allocation yo'q, HeapNumber bo'lib qolmaydi). 32-bitga chiqsa HeapNumber'ga aylanadi va bitwise operations sekinroq. **Asosiy printsip:** eng past indexli bit (eng o'ng) = eng yuqori priority. Aniq bit qiymatlarini eslab qolish shart emas — `mergeLanes`, `includesSomeLane`, `getHighestPriorityLane` kabi utility'lar bilan ishlash kerak.

**Mental model:**

```
Lane sifatini tekshirish:
- Past bit (Sync) tezroq — ko'p ish bo'lsa, oldinda
- Yuqori bit (Idle) sekinroq — boshqa ishlar bo'lmaganda

Bir vaqtda (SyncLane = bit 1, DefaultLane = bit 5):
fiber.lanes = 0b0000000000000000000000000100010
            = SyncLane | DefaultLane
            (sync va default — ikkalasi pending)
```

### Lane operations

```typescript
// React lane utilities (R18-R19 source, mental model)
const NoLanes = 0;
const SyncLane = 0b0000000000000000000000000000010;        // bit 1
const DefaultLane = 0b0000000000000000000000000100000;     // bit 5
const TransitionLane1 = 0b0000000000000000000000010000000; // bit 7
const IdleLane = 0b0100000000000000000000000000000;        // bit 29

// Qo'shish
let lanes = SyncLane;
lanes = mergeLanes(lanes, DefaultLane);  // lanes |= DefaultLane

// Tekshirish
if (includesSomeLane(lanes, SyncLane)) {
  // Sync update bor
}

// Olib tashlash
lanes = removeLanes(lanes, SyncLane);  // lanes &= ~SyncLane

// Eng yuqori priority
const highest = getHighestPriorityLanes(lanes);
// SyncLane, agar bor bo'lsa
// Aks holda InputContinuousLane, va h.k.
```

### Render Phase'da lanes

Reconciler `renderLanes` parametri bilan render qiladi — qaysi lane'lardagi update'larni ishlash:

```typescript
// React internal
function performWorkOnRoot(root) {
  // Eng yuqori priority lane'lar tanlanadi (ikkinchi argument — joriy
  // work-in-progress lanes, render davom etayotgan bo'lsa)
  const lanes = getNextLanes(root, NoLanes);
  
  if (lanes === NoLanes) return;
  
  // Render shu lane'lar uchun
  workInProgress = createWorkInProgress(root.current, lanes);
  
  while (workInProgress !== null) {
    if (shouldYield()) {
      // Yield — keyingi macrotask'da davom etish
      break;
    }
    workInProgress = performUnitOfWork(workInProgress);
  }
  
  // Render tugadi → commit
  if (workInProgress === null) {
    commitRoot(root, lanes);
  }
}
```

`getNextLanes(root)` — tree'dagi pending lane'lardan **eng yuqori priority**'larni tanlaydi. Past priority lane'lar keyingi render'larga qoldiriladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Lane utility funksiyalar:**

```typescript
// Manba: react-reconciler/src/ReactFiberLane.js (R18.3+/R19)
export const NoLanes = 0;
export const NoLane = 0;
export const SyncHydrationLane = 0b0000000000000000000000000000001;  // bit 0 (=1)
export const SyncLane = 0b0000000000000000000000000000010;            // bit 1 (=2)
export const InputContinuousHydrationLane = 0b0000000000000000000000000000100;  // bit 2 (=4)
export const InputContinuousLane = 0b0000000000000000000000000001000;           // bit 3 (=8)
export const DefaultHydrationLane = 0b0000000000000000000000000010000;          // bit 4 (=16)
export const DefaultLane = 0b0000000000000000000000000100000;                   // bit 5 (=32)
// ... va h.k.
// Eslatma: R18.0-R18.2 da SyncHydrationLane mavjud emas edi — SyncLane=1 edi.
// R18.3+ va R19 da SyncHydrationLane qo'shildi va bit pozitsiyalari surildi.

export function mergeLanes(a, b) {
  return a | b;
}

export function removeLanes(set, subset) {
  return set & ~subset;
}

export function intersectLanes(a, b) {
  return a & b;
}

export function includesSomeLane(a, b) {
  return (a & b) !== NoLanes;
}

export function isSubsetOfLanes(set, subset) {
  return (set & subset) === subset;
}

export function getHighestPriorityLane(lanes) {
  return lanes & -lanes;  // Past bit'ni ajratish (-x = ~x + 1)
}
```

**`getHighestPriorityLane` matematikasi:**

`lanes & -lanes` — eng past bit'ni qaytaradi (bitwise trick).

```
lanes = 0b00010100
-lanes = 0b11101100  (two's complement)

lanes & -lanes:
0b00010100
0b11101100
----------
0b00000100  ← eng past bit ajratildi
```

Bu — eng yuqori priority lane (lanes bitmap'ida past bit = yuqori priority).

**`getNextLanes` mexanikasi:**

```typescript
function getNextLanes(root, wipLanes) {
  const pendingLanes = root.pendingLanes;
  if (pendingLanes === NoLanes) return NoLanes;
  
  let nextLanes = NoLanes;
  
  // 1. Suspended lane'larni e'tiborga olish (Suspense bilan to'xtatilgan)
  const suspendedLanes = root.suspendedLanes;
  const pingedLanes = root.pingedLanes;
  
  // 2. Non-suspended lane'lardan eng yuqori priority
  const nonIdlePendingLanes = pendingLanes & NonIdleLanes;
  if (nonIdlePendingLanes !== NoLanes) {
    const nonIdleUnblockedLanes = nonIdlePendingLanes & ~suspendedLanes;
    if (nonIdleUnblockedLanes !== NoLanes) {
      nextLanes = getHighestPriorityLanes(nonIdleUnblockedLanes);
    } else {
      // Hammasi suspended — pinged'larni tanlash
      nextLanes = pingedLanes & nonIdlePendingLanes;
    }
  } else {
    // Idle lane'lar
    const unblockedLanes = pendingLanes & ~suspendedLanes;
    if (unblockedLanes !== NoLanes) {
      nextLanes = getHighestPriorityLanes(unblockedLanes);
    } else {
      nextLanes = pingedLanes;
    }
  }
  
  return nextLanes;
}
```

Bu funksiya har render boshida chaqiriladi — qaysi lane'lar bilan ishlashni aniqlash uchun.

**Lane masking:**

Tez-tez ishlatiladigan mask'lar:

```typescript
const SyncLane = 0b0000000000000000000000000000010;             // bit 1
const InputContinuousLane = 0b0000000000000000000000000001000;   // bit 3
const DefaultLane = 0b0000000000000000000000000100000;           // bit 5
const TransitionLanes = 0b0000000011111111111111110000000;       // bits 7..22 (16 ta lane)
const RetryLanes = 0b0000111100000000000000000000000;            // bits 23..26
const IdleLane = 0b0100000000000000000000000000000;              // bit 29

const SyncUpdateLanes = SyncLane | InputContinuousLane | DefaultLane;
const NonIdleLanes = SyncLane | InputContinuousHydrationLane
                   | InputContinuousLane | DefaultHydrationLane
                   | DefaultLane | TransitionLanes
                   | RetryLanes | SelectiveHydrationLane;
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Lanes mental model — bir Fiber'da bir nechta priority:

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  const [, startTransition] = useTransition();

  function handleMixed() {
    setCount(c => c + 1);  // SyncLane (urgent)
    
    startTransition(() => {
      setCount(c => c + 1);  // TransitionLane (low priority)
    });
    
    setCount(c => c + 1);  // SyncLane (urgent yana)
  }

  return <button onClick={handleMixed}>{count}</button>;
}

// handleMixed chaqirilganda Counter Fiber:
// fiber.lanes = SyncLane | TransitionLane
// 
// Render Phase:
// 1. getNextLanes → SyncLane (eng yuqori priority)
// 2. Render with SyncLane: setCount calls 1 va 3 ishlanadi
//    count = 0 → 2 (functional updater)
//    Commit
// 3. Render Phase yana: TransitionLane qoldi
// 4. Render with TransitionLane: setCount call 2 ishlanadi
//    count = 2 → 3
//    Commit
// 
// Foydalanuvchi 2 ta render ko'radi: 0 → 2 → 3
// Birinchi render — sync (darhol)
// Ikkinchi render — transition (uziluvchi, lekin tez kelish kerak)
```

Lanes va `useTransition`:

```tsx
function SearchInput() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<Item[]>([]);
  const [isPending, startTransition] = useTransition();

  function handleChange(value: string) {
    setQuery(value);  // SyncLane — input darhol updated
    
    startTransition(() => {
      // Bu callback ichidagi setState'lar TransitionLane
      setResults(searchHeavy(value));
    });
  }

  return (
    <>
      <input value={query} onChange={e => handleChange(e.target.value)} />
      {isPending && <Spinner />}  {/* TransitionLane render davom etayotgani */}
      <ResultList items={results} />
    </>
  );
}

// Tezda yozilganda:
// 1-keypress: query Fiber.lanes = SyncLane | TransitionLane
//   - getNextLanes → SyncLane
//   - Sync render: query=value, isPending=true
//   - Commit (input darhol yangilanadi)
//   - getNextLanes → TransitionLane
//   - Transition render boshlanadi (results)
// 
// 2-keypress (transition render davom etayotganida):
//   - Yangi query update keldi (SyncLane)
//   - Transition render TASHLANADI
//   - Sync render birinchi: yangi query
//   - Transition render qayta boshlandi: yangi value bilan
```

Background work — app level'da `IdleLane` to'g'ridan-to'g'ri ochilmagan:

```tsx
function App() {
  const [data, setData] = useState<Data | null>(null);

  useEffect(() => {
    // Background — analytics yuborish
    startTransition(() => {
      // TransitionLane — background work
      sendAnalytics(data);
    });
  }, [data]);

  return <MainContent data={data} />;
}

// Eslatma: React'da rasmiy "IdlePriority" API yo'q (app level'da)
// startTransition — Normal/Transition priority
// IdleLane — internal (offscreen, hydration, va h.k. uchun)
```

</details>

---

## Lane Types

### Nazariya

R18+'da bir nechta lane turi mavjud. Har lane mos priority bilan keladi:

| Lane | Priority | Qachon ishlatiladi |
|------|----------|---------------------|
| **`SyncLane`** | Eng yuqori | Click, keypress event handler ichidagi setState; legacy mode renders |
| **`InputContinuousLane`** | Yuqori | Drag, scroll, mousemove kabi continuous input'lar |
| **`DefaultLane`** | Normal | Default setState (event listener'siz, useEffect ichida, va h.k.) |
| **`TransitionLane` (1-16)** | Past | `useTransition`, `startTransition` ichidagi setState |
| **`RetryLane`** | Past | Suspense retry (lazy load qaytarish) |
| **`SelectiveHydrationLane`** | O'rta | R18 selective hydration (cross-ref `06-hydration.md`) |
| **`IdleLane`** | Eng past | Offscreen rendering, background task'lar |

**Hydration lane'lari:**

R18'da hydration uchun maxsus lane'lar:
- `InputContinuousHydrationLane` — input'da hydration
- `DefaultHydrationLane` — default hydration
- `IdleHydrationLane` — background hydration

Bu lane'lar selective hydration (Suspense bilan) uchun ishlatiladi (chuqur `06-hydration.md` da).

### TransitionLane'lar — 16 ta

`useTransition` har chaqirilganda **alohida** TransitionLane ishlatadi. Ya'ni — bir nechta transition bir vaqtda ishlay oladi (har biri o'z lane'sida):

```tsx
function App() {
  const [, startSearchTransition] = useTransition();
  const [, startFilterTransition] = useTransition();

  function handleSearch(q: string) {
    startSearchTransition(() => {  // TransitionLane1
      setSearchResults(search(q));
    });
  }

  function handleFilter(f: string) {
    startFilterTransition(() => {  // TransitionLane2
      setFilteredResults(filter(f));
    });
  }
}
```

16 ta TransitionLane mavjud. 17-transition uchun React eski lane'larni reuse qiladi (ko'pchilik holatlarda yetarli).

### Lane'lar va Concurrent Rendering

**Bir vaqtda ishlanish:**

```tsx
function Page() {
  const [count, setCount] = useState(0);
  const [searchResults, setSearchResults] = useState([]);
  const [, startTransition] = useTransition();

  function handleClick() {
    setCount(c => c + 1);  // SyncLane
  }

  function handleSearch(q: string) {
    startTransition(() => {
      setSearchResults(heavySearch(q));  // TransitionLane
    });
  }

  return (
    <>
      <button onClick={handleClick}>Count: {count}</button>
      <input onChange={e => handleSearch(e.target.value)} />
      <Results items={searchResults} />
    </>
  );
}

// Foydalanuvchi click qilganda:
// Page Fiber.lanes = SyncLane
// Render with SyncLane
// Commit (count yangilandi)

// Foydalanuvchi yozayotganda:
// Page Fiber.lanes = SyncLane | TransitionLane
// 1. Render with SyncLane: setCount uchun (lekin yo'q — faqat search)
//    Aslida sync update yo'q bo'lsa, scheduler keyingi priority'ga o'tadi
// 2. Render with TransitionLane: searchResults
// 3. Commit

// Foydalanuvchi yozayotgan paytda click qildi:
// Page Fiber.lanes = SyncLane | TransitionLane
// 1. Transition render to'xtaydi (sync update keldi)
// 2. Sync render: count
// 3. Commit
// 4. Transition render qayta boshlanadi
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Lane qiymatlari (R18 source):**

```typescript
export const TotalLanes = 31;

export const NoLanes: Lanes = /*                        */ 0b0000000000000000000000000000000;
export const NoLane: Lane = /*                          */ 0b0000000000000000000000000000000;

export const SyncHydrationLane: Lane = /*               */ 0b0000000000000000000000000000001;
export const SyncLane: Lane = /*                        */ 0b0000000000000000000000000000010;

export const InputContinuousHydrationLane: Lane = /*    */ 0b0000000000000000000000000000100;
export const InputContinuousLane: Lane = /*             */ 0b0000000000000000000000000001000;

export const DefaultHydrationLane: Lane = /*            */ 0b0000000000000000000000000010000;
export const DefaultLane: Lane = /*                     */ 0b0000000000000000000000000100000;

const TransitionHydrationLane: Lane = /*                */ 0b0000000000000000000000001000000;
const TransitionLanes: Lanes = /*                       */ 0b0000000011111111111111110000000;
const TransitionLane1: Lane = /*                        */ 0b0000000000000000000000010000000;
// ... TransitionLane2 - TransitionLane16

const RetryLanes: Lanes = /*                            */ 0b0000111100000000000000000000000;

export const SelectiveHydrationLane: Lane = /*          */ 0b0001000000000000000000000000000;

const NonIdleLanes: Lanes = /*                          */ 0b0001111111111111111111111111111;

export const IdleHydrationLane: Lane = /*               */ 0b0010000000000000000000000000000;
export const IdleLane: Lane = /*                        */ 0b0100000000000000000000000000000;

export const OffscreenLane: Lane = /*                   */ 0b1000000000000000000000000000000;
```

> Bit qiymatlari R18-R19 source kodidan olingan. Yangi versiyalarda lane'lar qo'shilishi yoki o'zgarishi mumkin.

**TransitionLane allocation:**

```typescript
let nextTransitionLane = TransitionLane1;

function claimNextTransitionLane(): Lane {
  const lane = nextTransitionLane;
  nextTransitionLane <<= 1;  // Bitni siljitish (keyingi lane'ga)
  
  if ((nextTransitionLane & TransitionLanes) === NoLanes) {
    // Barcha 16 ta lane ishlatildi — boshidan
    nextTransitionLane = TransitionLane1;
  }
  
  return lane;
}

function startTransition(scope) {
  const previousTransition = ReactCurrentBatchConfig.transition;
  ReactCurrentBatchConfig.transition = {};
  
  // Yangi transition lane
  const currentTransition = ReactCurrentBatchConfig.transition;
  currentTransition._updatedFibers = new Set();
  
  try {
    scope();
  } finally {
    ReactCurrentBatchConfig.transition = previousTransition;
  }
}
```

**Lane priority hierarchy:**

```typescript
function getHighestPriorityLanes(lanes) {
  switch (getHighestPriorityLane(lanes)) {
    case SyncHydrationLane:
      return SyncHydrationLane;
    case SyncLane:
      return SyncLane;
    case InputContinuousHydrationLane:
      return InputContinuousHydrationLane;
    case InputContinuousLane:
      return InputContinuousLane;
    case DefaultHydrationLane:
      return DefaultHydrationLane;
    case DefaultLane:
      return DefaultLane;
    case TransitionHydrationLane:
      return TransitionHydrationLane;
    case TransitionLane1:
    case TransitionLane2:
    // ... barcha TransitionLane'lar
      return lanes & TransitionLanes;  // Barcha transition'lar birga
    case RetryLane1:
    case RetryLane2:
    // ...
      return lanes & RetryLanes;
    case SelectiveHydrationLane:
      return SelectiveHydrationLane;
    case IdleHydrationLane:
      return IdleHydrationLane;
    case IdleLane:
      return IdleLane;
    case OffscreenLane:
      return OffscreenLane;
    default:
      return lanes;
  }
}
```

Ko'p TransitionLane'lar **birga** rejalashtiriladi (group). Bu — bir nechta transition bir xil priority darajasida deb hisoblanadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Multiple transitions misol:

```tsx
function MultiTransition() {
  const [searchResults, setSearchResults] = useState<Item[]>([]);
  const [filterResults, setFilterResults] = useState<Item[]>([]);
  const [, startSearchTransition] = useTransition();
  const [, startFilterTransition] = useTransition();

  function handleSearchInput(q: string) {
    startSearchTransition(() => {
      // TransitionLane1
      setSearchResults(heavySearch(q));
    });
  }

  function handleFilterChange(f: string) {
    startFilterTransition(() => {
      // TransitionLane2 (alohida)
      setFilterResults(heavyFilter(f));
    });
  }

  return (
    <>
      <input onChange={e => handleSearchInput(e.target.value)} placeholder="Search" />
      <select onChange={e => handleFilterChange(e.target.value)}>
        <option value="all">All</option>
        <option value="active">Active</option>
      </select>
      <Results search={searchResults} filter={filterResults} />
    </>
  );
}

// Search yangilandi va Filter yangilandi:
// fiber.lanes = TransitionLane1 | TransitionLane2
// Reconciler ikkalasini birga ishlaydi (bir render'da)
// Commit
```

Lanes interaction visualization:

```tsx
function Demo() {
  const [count, setCount] = useState(0);
  const [data, setData] = useState<Data[]>([]);
  const [, startTransition] = useTransition();

  function handleClick() {
    // SyncLane — urgent
    setCount(c => c + 1);
  }

  function loadData() {
    startTransition(() => {
      // TransitionLane — background
      const newData = expensiveLoad();
      setData(newData);
    });
  }

  return (
    <>
      <button onClick={handleClick}>Count: {count}</button>
      <button onClick={loadData}>Load Data</button>
      <DataList items={data} />
    </>
  );
}

// Scenario 1: Avval Click, keyin Load
// 1. Click → SyncLane → Render → Commit (instant)
// 2. Load → TransitionLane → Render (uziluvchi) → Commit (eventually)

// Scenario 2: Avval Load, keyin Click (Load davom etayotgan paytda)
// 1. Load → TransitionLane → Render boshlandi
// 2. Click keldi → SyncLane (yuqori priority)
// 3. Transition render TASHLANADI
// 4. Sync render boshlanadi → Commit (instant)
// 5. Transition render qayta boshlanadi
// 6. Eventual Commit

// Scenario 3: Click + Load birga (bir handler ichida)
// fiber.lanes = SyncLane | TransitionLane
// 1. Sync render birinchi: count update
// 2. Commit (count yangilangan)
// 3. Transition render: data update
// 4. Commit (data yangilangan)
// → 2 ta commit, ikkalasi ham vizual ko'rinadi
```

</details>

---

## Time Slicing va Frame Budget

### Nazariya

**Time slicing** — render ishini kichik **chunk**'larga (~5ms) bo'lib, har chunk orasida browser'ga yo'l berish (yield). Bu — concurrent rendering'ning amaliy mexanizmi.

**Frame budget:**

Browser 60 fps'da har frame 16.67ms ichida tugashi kerak. Bu vaqt JavaScript ijrosi (sizning kodingiz, React) va browser'ning o'z ishi (style recalculation, layout, paint, composite) orasida bo'linadi. JavaScript qancha uzoq ishlasa, browser'ga style/layout/paint uchun shuncha kam vaqt qoladi.

React Scheduler **5ms ish budget**ni o'rnatadi (`frameYieldMs = 5`). Har 5ms ish qilgandan keyin yield qilinadi — browser'ga rendering ishi uchun vaqt qoladi. Bu 5ms — frame'dan ancha kichik, shuning uchun bitta frame ichida React bir necha marta yield qilib, browser bilan navbatlashishi mumkin.

> **Eslatma:** Eski versiya React'da budget kattaroq edi; R18'da `frameYieldMs = 5` belgilandi (manba: `scheduler/src/forks/Scheduler.js`). Scheduler shu fayl ichida yana ikkita konstantani saqlaydi: `continuousYieldMs = 50` (input bo'lmaganda budget'ni cho'zish oraliq chegarasi) va `maxYieldMs = 300` (yield qilishni kechiktirib bo'lmaydigan absolyut chegara). Bular `isInputPending()`-based optimization uchun ishlatiladi.

### Time slicing mexanikasi

```typescript
// Scheduler internal (soddalashtirilgan)
const frameYieldMs = 5;
let startTime = -1;

function shouldYieldToHost() {
  const timeElapsed = getCurrentTime() - startTime;
  if (timeElapsed < frameYieldMs) {
    return false;  // Hali budget bor
  }
  return true;  // Yield kerak
}

function performWorkUntilDeadline() {
  if (scheduledHostCallback !== null) {
    const currentTime = getCurrentTime();
    startTime = currentTime;
    
    const hasMoreWork = scheduledHostCallback(currentTime);
    
    if (hasMoreWork) {
      // Yana ish bor — keyingi macrotask'da davom
      schedulePerformWorkUntilDeadline();
    }
  }
}

function workLoop(initialTime) {
  let currentTask = peek(taskQueue);
  
  while (currentTask !== null) {
    if (currentTask.expirationTime > getCurrentTime() && shouldYieldToHost()) {
      // Yield
      break;
    }
    
    // Task ishlanadi
    const callback = currentTask.callback;
    currentTask.callback = null;
    
    const didTimeout = currentTask.expirationTime <= getCurrentTime();
    const continuation = callback(didTimeout);
    
    if (typeof continuation === 'function') {
      currentTask.callback = continuation;
    } else {
      pop(taskQueue);
    }
    
    currentTask = peek(taskQueue);
  }
  
  return currentTask !== null;  // Hali ish bormi
}
```

### Browser rendering pipeline va React Scheduler

```
Macrotask 1 (React work):
┌──────────────────────────────────────┐
│  performWorkUntilDeadline            │
│    workLoop (5ms budget)             │
│      Fiber 1, 2, 3, ..., N           │
│      shouldYield → true (5ms)        │
│    schedulePerformWorkUntilDeadline  │
│      port.postMessage(null)          │
└──────────────────────────────────────┘
          ↓
Microtask queue flush
          ↓
Browser rendering:
┌──────────────────────────────────────┐
│  rAF callbacks                       │
│  Style recalculation                 │
│  Layout                              │
│  Paint                               │
│  Composite                           │
└──────────────────────────────────────┘
          ↓
Macrotask 2 (React davom):
┌──────────────────────────────────────┐
│  port1.onmessage → performWorkUntilDeadline │
│    workLoop (yangi 5ms budget)       │
│      Fiber N+1, N+2, ...             │
└──────────────────────────────────────┘
```

Har macrotask orasida browser paint qila oladi — UI doim javob beradi.

### Time slicing va Strict Mode

Strict Mode dev mode'da render funksiyasini va effect'larni ikki marta chaqiradi (purity va cleanup'ni tekshirish uchun). Shu sababli dev'da har update'da bajariladigan ish ikki barobar — bu time slicing ko'rsatkichlarini (chunk soni, jami render vaqti) sun'iy oshiradi. Production build'da Strict Mode no-op, bu ikki barobar ish yo'q, shuning uchun profiling production build'da o'tkazilishi kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

**Scheduler yield budget tarixi (`packages/scheduler/src/forks/Scheduler.js`):**

| React versiya | Yield budget | Eslatma |
|---------------|--------------|---------|
| R16.0–R16.4 | rAF-based, kattaroq frame budget | Dastlabki Fiber — `requestAnimationFrame` orqali frame deadline |
| R16.5–R17.x | `yieldInterval = 5ms` | Scheduler alohida paketga ko'chgan, fixed 5ms budget |
| R18+ | `frameYieldMs = 5ms` | Bir xil 5ms, lekin `isInputPending()` bilan kengaytirildi (continuousYieldMs=50ms, maxYieldMs=300ms — input yo'qsa cho'zish mumkin) |

R18'da budget raqami **o'zgarmadi** (5ms qoldi); o'zgarish — `isInputPending` Chrome API bilan dynamic budget extension qo'shildi. Input yo'q vaqtda React yield qilmasdan ko'proq ishlay oladi (kerakmas yield kamaytirish).

**Continuation pattern:**

`workLoop` to'liq tugamasdan yield qilganda, React `workInProgress` o'zgaruvchisini saqlaydi. Keyingi macrotask'da yangi `workLoop` chaqirilganda, shu joydan davom etadi:

```typescript
let workInProgress: Fiber | null = null;

function performWorkOnRoot(root, lanes) {
  if (workInProgress === null) {
    // Yangi render — workInProgress yaratish
    workInProgress = createWorkInProgress(root.current, null);
  }
  
  // Existing workInProgress'dan davom
  while (workInProgress !== null) {
    if (shouldYield()) break;
    workInProgress = performUnitOfWork(workInProgress);
  }
  
  // Hali tugamagan
  if (workInProgress !== null) {
    return RootInProgress;
  }
  
  // Render tugadi → commit
  commitRoot(root, lanes);
  return RootCompleted;
}
```

**Yield budget va high-priority interrupts:**

`shouldYield()` — faqat **vaqt budget** ni tekshirmaydi. U input bo'lganmi yo'qmi ham tekshiradi:

```typescript
function shouldYieldToHost() {
  const timeElapsed = getCurrentTime() - frameStartTime;
  
  if (timeElapsed < frameYieldMs) {
    return false;  // Budget bor
  }
  
  // Budget tugadi — kuzatish
  if (enableIsInputPending) {
    if (needsPaint || scheduling.isInputPending()) {
      return true;  // Input bor → yield
    }
    
    // Input yo'q — biroz cho'zish mumkin (uzoq budget)
    if (timeElapsed < continuousYieldMs) {
      return false;
    }
    
    if (timeElapsed < maxYieldMs) {
      return scheduling.isInputPending({ includeContinuous: true });
    }
  }
  
  return true;
}
```

`navigator.scheduling.isInputPending()` — yangi (Chrome 87+) browser API. Pending input bor-yo'qligini tekshiradi. Yo'q bo'lsa — React budget'ni cho'zishi mumkin (`continuousYieldMs = 50` ms oraliq threshold, `maxYieldMs = 300` ms absolute ceiling). Bu — performance optimization (kerakmas yield kamaytirish). Eslatma: `enableIsInputPending` Scheduler feature flag ostida — production React'da default `false`, eksperimental optimization sifatida turibdi.

Boshqa browser'larda (Safari, Firefox) bu API yo'q — fallback 5ms budget.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Time slicing kuzatish:

```tsx
function HeavyList() {
  const [items, setItems] = useState<Item[]>([]);
  const [renderTime, setRenderTime] = useState<number[]>([]);

  function loadItems() {
    const start = performance.now();
    setItems(generateLargeArray(10000));
    
    requestAnimationFrame(() => {
      const end = performance.now();
      setRenderTime(times => [...times, end - start]);
    });
  }

  return (
    <>
      <button onClick={loadItems}>Load (sync)</button>
      <p>Last render: {renderTime[renderTime.length - 1]?.toFixed(0)}ms</p>
      <ul>
        {items.slice(0, 100).map(i => <li key={i.id}>{i.name}</li>)}
      </ul>
    </>
  );
}

// Sync rendering: ~800ms (UI muzlaydi)
// Concurrent + useTransition: chunked (har 5ms da yield)
```

useTransition bilan time slicing:

```tsx
function ConcurrentList() {
  const [items, setItems] = useState<Item[]>([]);
  const [isPending, startTransition] = useTransition();

  function loadItems() {
    startTransition(() => {
      setItems(generateLargeArray(10000));
    });
  }

  return (
    <>
      <button onClick={loadItems}>
        Load (concurrent)
      </button>
      {isPending && <p>Yangilanmoqda...</p>}
      <ul>
        {items.slice(0, 100).map(i => <li key={i.id}>{i.name}</li>)}
      </ul>
    </>
  );
}

// Concurrent rendering:
// 1. Click → setState (TransitionLane)
// 2. isPending = true (instant UI feedback)
// 3. Render boshlanadi — har 5ms da yield
// 4. Browser rasmlarini ko'rsatishda davom etadi
// 5. Render tugagach — Commit
// 6. isPending = false
```

</details>

---

## Interruptible Rendering

### Nazariya

**Interruptible rendering** — concurrent rendering'ning markaziy g'oyasi. Render davom etayotgan paytda **yangi update kelganda**, React eski render'ni **tashlaydi** va yangi update bilan qaytadan boshlaydi.

**Scenario:**

```tsx
function App() {
  const [searchQuery, setSearchQuery] = useState('');
  const [, startTransition] = useTransition();

  function handleChange(value: string) {
    setSearchQuery(value);  // SyncLane
    
    startTransition(() => {
      // TransitionLane
      const results = expensiveSearch(value);
      setSearchResults(results);
    });
  }
}

// Foydalanuvchi tezda yozadi: "a", "ab", "abc"
//
// 1. "a" — startTransition: render boshlanadi (TransitionLane)
//    - 5ms ishladi, yield
//    - 5ms ishladi, yield (rendering davom etayapti)
// 
// 2. "ab" keldi — yangi TransitionLane update
//    - Eski "a" render TASHLANDI
//    - workInProgress tree butunlay tashlanadi
//    - Yangi render "ab" uchun boshlanadi
// 
// 3. "abc" keldi
//    - "ab" render tashlanadi
//    - "abc" render boshlanadi
// 
// 4. Foydalanuvchi to'xtagach — render tugaydi va commit
```

**Tashlanuvchi render — nima bo'ladi:**

- `workInProgress` tree butunlay tashlanadi
- `current` tree daxlsiz (foydalanuvchi eski UI'ni ko'rishda davom etadi)
- Hech qanday DOM mutation amalga oshmagan
- Hech qanday effect ishga tushmagan
- Render davomida hisoblash qilingan (komponent funksiyalar chaqirilgan, hooks bajarilgan) — lekin natija **yo'qoladi**

**Shu sababli render PURE bo'lishi shart:**

Agar render davomida side effect bo'lsa (mutation, fetch, va h.k.), ular yo'qolmaydi (browser/network ish bajarilgan). Render tashlansa — bu side effect "yarim ish" bo'lib qoladi.

### Restart va priority interaction

```
Render Phase priority hierarchy:
SyncLane > InputContinuousLane > DefaultLane > TransitionLane > IdleLane

Misol:
1. Foydalanuvchi click qildi → SyncLane render boshlandi
2. Render davom etayotganda useTransition setState chaqirildi → TransitionLane
3. Sync render PRIORITY YUQORI → davom etadi
4. Sync render tugadi → Commit
5. Endi TransitionLane render boshlanadi
```

```
Boshqa misol:
1. useTransition setState chaqirildi → TransitionLane render boshlandi
2. Render bir necha chunk davom etdi (har 5ms da yield qildi)
3. Foydalanuvchi click qildi → SyncLane (yuqori priority)
4. TransitionLane render TASHLANADI
5. Sync render boshlanadi (click handler ichidagi setState)
6. Sync render tugadi → Commit
7. TransitionLane render qayta boshlanadi (yangi state bilan)
```

### Restart vs Suspend

**Restart** — render TASHLANADI va qaytadan boshlanadi (yangi update sababli).

**Suspend** — render TO'XTATILADI va keyinroq DAVOM ettiriladi (resource yuklanguncha — Suspense bilan).

Bular bir-biridan farqli mexanizmlar:
- Restart — yangi state bilan
- Suspend — bir xil state, lekin async resource kutilmoqda

<details>
<summary><strong>Under the Hood</strong></summary>

**Restart implementation:**

```typescript
// react-reconciler/src/ReactFiberWorkLoop.js (soddalashtirilgan)
function renderRootConcurrent(root, lanes) {
  // Davom etayotgan render boshqa root yoki boshqa lane'lar uchun bo'lsa —
  // stack'ni tashlab, yangi workInProgress tree'dan boshlash
  if (workInProgressRoot !== root || workInProgressRootRenderLanes !== lanes) {
    prepareFreshStack(root, lanes);  // workInProgress = createWorkInProgress(root.current, null)
  }

  // Render loop
  while (workInProgress !== null) {
    if (shouldYield()) break;
    workInProgress = performUnitOfWork(workInProgress);
  }
}
```

`prepareFreshStack` har **restart**'da yangi workInProgress tree'ni `createWorkInProgress(root.current, null)` orqali quradi. Yangi tree alternate slot'ni qayta ishlatadi — oldingi yarim qurilgan workInProgress'ning natijasi tashlanadi, `current` tree esa tegilmaydi.

**Restart sabab:**

1. **Yangi update keldi** — yangi state bilan render kerak
2. **Priority o'zgardi** — yangi yuqori priority update keldi
3. **Lane priority shifted** — bir nechta lane'larda update bo'lsa, eng yuqori priority birinchi

**Restart vs cancel:**

- **Restart** — work davom etadi (yangi state bilan)
- **Cancel** — work butunlay tashlanadi (yangi update yo'q)

Cancel — `unmount` paytida bo'ladi:

```typescript
function commitDeletionEffects(root, parentFiber, deletedFiber) {
  // 1. Pending lane'lardan deleted fiber'ni olib tashlash
  // 2. Effect'larni cleanup qilish
  // 3. DOM'dan o'chirish
}
```

**Restart ko'p marta — performance:**

Foydalanuvchi tezda yozayotganda, har keystroke restart keltiradi. Ya'ni — render bir nechta marta boshlanib tashlangan (waste). React jamoasi bu trade-off'ni qabul qilgan:

- Per-keystroke render — O(n) ish (faqat workInProgress tree quriladi)
- Foydalanuvchi to'xtagach — bitta to'liq render commit qilinadi
- UI doim javob beradi

Bu — "wasted work" — performance optimization vs responsivity trade-off. Concurrent rendering'da responsivity prioritized.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Restart visualization:

```tsx
function SearchDemo() {
  const [query, setQuery] = useState('');
  const [renderCount, setRenderCount] = useState(0);
  const [isPending, startTransition] = useTransition();

  function handleChange(value: string) {
    setQuery(value);
    
    startTransition(() => {
      // Heavy work
      setRenderCount(c => c + 1);
      const results = expensiveOperation(value);
      console.log('Render attempt for:', value);
    });
  }

  return (
    <>
      <input value={query} onChange={e => handleChange(e.target.value)} />
      {isPending && <p>...</p>}
      <p>Render count: {renderCount}</p>
    </>
  );
}

// Tezda "abc" yozildi:
// Console:
// "Render attempt for: a"   ← boshlandi
// "Render attempt for: ab"  ← restart
// "Render attempt for: abc" ← restart yana
// 
// renderCount commit qilinmagan render'lar uchun ham oshadi (+ functional updater chaqiriladi)
// Lekin commit faqat oxirgi muvaffaqiyatli render uchun bo'ladi
```

Restart trade-off:

```tsx
function ExpensiveTransition() {
  const [value, setValue] = useState('');
  const [, startTransition] = useTransition();

  function handleChange(newValue: string) {
    startTransition(() => {
      setValue(newValue);
      // Heavy computation har transition render davomida
      heavyComputation(newValue);
    });
  }

  return <input onChange={e => handleChange(e.target.value)} />;
}

// Foydalanuvchi 10 keystroke yozdi:
// 10 ta startTransition → 10 ta render boshlanishi
// Lekin 9 tasi RESTART bilan tashlanadi
// Faqat 10-render commit qilinadi
// 
// Heavy computation 10 marta chaqiriladi (har boshlanishida)
// Bu "wasted work"
// 
// Optimization: useDeferredValue + useMemo
function Better() {
  const [value, setValue] = useState('');
  const deferredValue = useDeferredValue(value);
  
  // useMemo cache qiladi — deferredValue o'zgarmaguncha heavyComputation
  // qayta ishlamaydi (transition restart bo'lsa ham)
  const result = useMemo(
    () => heavyComputation(deferredValue),
    [deferredValue]
  );

  return <Result data={result} />;
}
```

</details>

---

## Expiration va Starvation Prevention

### Nazariya

**Starvation** — low-priority task'ning **hech qachon ishga tushmasligi**. Misol:
- Foydalanuvchi cheksiz tezda yozayotgan (sync update'lar)
- Transition render har safar restart bilan tashlanadi
- Transition update **never commits** → starvation

React Scheduler **expiration** mexanizmi orqali starvation'ni oldini oladi. Har lane'ning **expirationTime**'i bor — agar shu vaqtdan o'tgach hali ishga tushmagan bo'lsa, **sync** sifatida bajariladi (boshqa update'larni kutmasdan).

### ExpirationTime hisob-kitobi

```typescript
// react-reconciler/src/ReactFiberLane.js (soddalashtirilgan)
function computeExpirationTime(lane, currentTime) {
  switch (lane) {
    case SyncLane:
    case InputContinuousLane:
      // Yuqori priority — short timeout
      return currentTime + 250;

    case DefaultLane:
      // Normal priority — medium timeout
      return currentTime + 5000;

    case TransitionLane1:
    case TransitionLane2:
    // ... barcha TransitionLane'lar
      // Past priority — long timeout
      return currentTime + 5000;

    case IdleLane:
    case OffscreenLane:
      // Eng past priority — hech qachon expire bo'lmaydi
      return NoTimestamp;

    default:
      return NoTimestamp;
  }
}
```

**Mental model:**

```
Lane:           Default expiration:
SyncLane        250ms
DefaultLane     5000ms (5 sekund)
TransitionLane  5000ms (5 sekund)
IdleLane        ∞ (cheksiz)
```

Lane qancha vaqt kutsa — `expirationTime` yaqinlashadi. `getCurrentTime() >= expirationTime` bo'lsa, lane **expired** deb belgilanadi.

### Expired lane'larni ishlash

```typescript
function performWorkOnRoot(root) {
  const lanes = getNextLanes(root, NoLanes);
  
  // Expired lane'larni tekshirish
  if (includesExpiredLane(root, lanes)) {
    // Sync rejimida ishlash (yield qilmasdan)
    exitStatus = renderRootSync(root, lanes);
  } else {
    // Concurrent rejim (yield bilan)
    exitStatus = renderRootConcurrent(root, lanes);
  }
  
  // Commit
  commitRoot(root, lanes);
}

function markStarvedLanesAsExpired(root, currentTime) {
  const pendingLanes = root.pendingLanes;
  const expirationTimes = root.expirationTimes;
  
  let lanes = pendingLanes;
  while (lanes > 0) {
    const index = pickArbitraryLaneIndex(lanes);
    const lane = 1 << index;
    
    const expirationTime = expirationTimes[index];
    if (expirationTime === NoTimestamp) {
      // Hali expiration vaqti hisoblangan emas
      expirationTimes[index] = computeExpirationTime(lane, currentTime);
    } else if (expirationTime <= currentTime) {
      // Expired — sync sifatida belgilash
      root.expiredLanes |= lane;
    }
    
    lanes &= ~lane;
  }
}
```

**Expired bo'lganda:**

1. Lane `root.expiredLanes`'ga qo'shiladi
2. Render Phase **`renderRootSync`** ishlatadi (yield qilmasdan)
3. Bu render TASHLANMAYDI (yangi update kelsa ham, expired sync render davom etadi)
4. Commit normal yo'l bilan

### Starvation prevention misol

```tsx
function App() {
  const [highCount, setHighCount] = useState(0);
  const [lowCount, setLowCount] = useState(0);
  const [, startTransition] = useTransition();

  // TransitionLane'dan yuqori priority — har 100ms
  useEffect(() => {
    const id = setInterval(() => {
      setHighCount(c => c + 1);  // DefaultLane (setInterval — React event handler emas)
    }, 100);
    return () => clearInterval(id);
  }, []);

  // Past priority — bir marta
  useEffect(() => {
    startTransition(() => {
      setLowCount(c => c + 1);  // TransitionLane
    });
  }, []);

  return <div>High: {highCount}, Low: {lowCount}</div>;
}

// Scenario:
// - High count har 100ms → DefaultLane render (setInterval callback — `getCurrentUpdatePriority`
//   `DefaultEventPriority` qaytaradi; SyncLane faqat discrete event handler yoki flushSync uchun)
// - Low count update boshida bir marta → TransitionLane render
//
// Har 100ms da DefaultLane render boshlanadi → TransitionLane restart bilan tashlanadi
// 
// Agar starvation prevention bo'lmasa:
// - Low count hech qachon commit bo'lmasdi
// 
// React'ning yechimi:
// - 5 sekund o'tgach (TransitionLane expirationTime) → expired deb belgilanadi
// - Keyingi render'da TransitionLane sync sifatida ishlanadi
// - High count update bilan birga TransitionLane ham commit bo'ladi
// - Low count yangilandi
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Lane expiration tracking:**

```typescript
// FiberRoot structure
type FiberRoot = {
  // ...
  expirationTimes: Array<number>,  // Lane index → expirationTime
  pendingLanes: Lanes,
  expiredLanes: Lanes,
  // ...
};
```

`expirationTimes` — 31-element array (har bit uchun bitta). Har lane uchun expiration time alohida saqlanadi.

**`markRootExpired`:**

```typescript
function markRootExpired(root, lanes) {
  root.expiredLanes |= lanes & root.pendingLanes;
}

function includesExpiredLane(root, lanes) {
  return (root.expiredLanes & lanes) !== NoLanes;
}
```

**Sync render bilan expired ishlash:**

```typescript
function renderRootSync(root, lanes) {
  // workInProgress yaratish
  prepareFreshStack(root, lanes);
  
  // Sync loop — shouldYield TEKSHIRILMAYDI
  do {
    try {
      workLoopSync();
      break;
    } catch (e) {
      handleError(root, e);
    }
  } while (true);
  
  return RootCompleted;
}

function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
    // shouldYield() YO'Q — to'liq ishlanadi
  }
}
```

Sync render — interruptible emas. Boshlandimi, to'liq tugaguncha bajariladi. Bu — expired lane'lar uchun ishlatiladi (boshqa lane'lar bilan kompromiss qilmaslik).

**Expiration tweaking:**

Past versiyalarda expiration vaqti boshqacha edi:

```
R16-R17:
- LowPriority: 5000ms
- IdlePriority: never

R18+:
- DefaultLane: 5000ms
- TransitionLane: 5000ms
- IdleLane: never (yoki `maxSigned31BitInt = 1073741823` — ≈12.4 kun)
```

Ko'p hollarda expiration ishga tushmaydi — render odatda vaqtida tugaydi. Lekin starvation prevention kafolat sifatida bor.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Starvation simulation:

```tsx
function StarvationDemo() {
  const [searchResults, setSearchResults] = useState<Item[]>([]);
  const [, startTransition] = useTransition();
  const [tickCount, setTickCount] = useState(0);

  // DefaultLane update har 50ms — TransitionLane'ni starve qilishga harakat
  useEffect(() => {
    const id = setInterval(() => {
      setTickCount(c => c + 1);  // DefaultLane (setInterval — React event handler emas)
    }, 50);
    return () => clearInterval(id);
  }, []);

  function handleSearch() {
    startTransition(() => {
      // Heavy work — transition
      setSearchResults(expensiveSearch());
    });
  }

  return (
    <>
      <p>Tick: {tickCount}</p>
      <button onClick={handleSearch}>Search</button>
      <p>Results: {searchResults.length}</p>
    </>
  );
}

// Foydalanuvchi "Search" bosadi:
// 1. TransitionLane render boshlanadi
// 2. Tick interval 50ms — DefaultLane update keladi (yuqori priority)
// 3. TransitionLane render TASHLANADI
// 4. DefaultLane render → commit
// 5. TransitionLane render qayta boshlanadi
// 6. Yana 50ms — yana tashlanadi
// 7. ... bu cycle davom etadi
// 
// 5 sekund o'tgach:
// 8. TransitionLane EXPIRED
// 9. Keyingi render'da sync sifatida bajariladi
// 10. Commit (results yangilandi)
// 
// Foydalanuvchi 5 sekund kechikishni sezadi (lekin starvation YO'Q)
```

Real-world starvation prevention:

```tsx
// Banking app — har sekund balance update (DefaultLane)
// Background — transaction history fetch (TransitionLane)
function BankApp() {
  const [balance, setBalance] = useState(0);
  const [transactions, setTransactions] = useState<Transaction[]>([]);
  const [, startTransition] = useTransition();

  // Real-time balance polling
  useEffect(() => {
    const id = setInterval(() => {
      fetchBalance().then(setBalance);  // DefaultLane (promise callback ichida)
    }, 1000);
    return () => clearInterval(id);
  }, []);

  // Background transaction load
  useEffect(() => {
    fetchTransactions().then(t => {
      // startTransition scope sync tugaydi; promise resolve bo'lganda
      // transition flag o'chgan bo'ladi. Transition lane uchun setState
      // aynan .then ichida startTransition bilan qayta o'ralishi kerak
      startTransition(() => {
        setTransactions(t);  // TransitionLane
      });
    });
  }, []);

  return (
    <>
      <h1>Balance: ${balance}</h1>
      <TransactionList items={transactions} />
    </>
  );
}

// Balance update tez-tez (DefaultLane). Transactions — TransitionLane.
// DefaultLane TransitionLane'dan yuqori priority, shuning uchun agar balance
// updates juda tez-tez kelsa, transactions render takror restart bilan
// tashlanishi mumkin edi.
// 
// Lekin starvation prevention bor — TransitionLane expirationTime (5 sekund)
// o'tgach expired deb belgilanadi va sync sifatida commit qilinadi.
// Foydalanuvchi har holda transactions ko'radi.
```

</details>

---

## MessageChannel vs requestIdleCallback

### Nazariya

React Scheduler yield mexanizmi sifatida **`MessageChannel`**'ni tanlagan. Eski versiyada `requestIdleCallback` ishlatilgan edi, lekin almashtirilgan.

### `requestIdleCallback` muammolari

`requestIdleCallback` — Chrome 47'da (2015-yil dekabr) chiqarilgan API. Browser "idle" deb hisoblagan paytda callback chaqiradi:

```typescript
window.requestIdleCallback((deadline) => {
  // deadline.timeRemaining() — qancha vaqt qoldi (ms)
  while (deadline.timeRemaining() > 0 && hasWork()) {
    doWork();
  }
});
```

**Muammolari:**

1. **Cross-browser support yo'q** — Safari `requestIdleCallback`'ni hech qachon shipped qilmagan (WebKit'da hali ham yo'q). React Fiber dizayni bo'lib turgan paytda (2017) ham yo'q edi.
2. **Frequency past** — callback faqat browser "idle" deb hisoblagan paytda chaqiriladi. Idle period kelmaguncha kutiladi, shuning uchun chaqiruv chastotasi past va animation/responsiveness uchun yetarli emas.
3. **Deterministic emas** — "idle" deb hisoblangan vaqt implementation-defined; React render boshlanishi browser ixtiyoriga bog'lanib qoladi.
4. **Polyfill aniq emas** — yo'q browser'larda nima bilan almashtirish noaniq.

### `MessageChannel` afzalligi

```typescript
const channel = new MessageChannel();
const port = channel.port2;

channel.port1.onmessage = function performWork() {
  // Work bajariladi
};

function scheduleWork() {
  port.postMessage(null);  // Yangi macrotask rejalashtirish
}
```

**Afzalliklari:**

1. **Universal support** — barcha modern browserlar
2. **Frequency yuqori** — har macrotask'dan keyin ishlaydi (har frame'da bir nechta marta mumkin)
3. **Deterministic** — `postMessage` chaqirilganda darhol macrotask rejalashtiriladi
4. **Browser rendering pipeline bilan integration** — macrotask orasida browser rAF, paint, layout qila oladi

### MessageChannel ishlash printsipi

```
JS macrotask 1:
  performWork:
    workLoop (5ms)
    shouldYield → true
    port.postMessage(null)  ← yangi macrotask rejalashtirish
                             (bu macrotask oxirida)
                             
Microtask queue ishlanadi (Promise.then, queueMicrotask)

Browser rendering pipeline:
  rAF callbacks
  Style recalc
  Layout
  Paint
  Composite

JS macrotask 2:
  port1.onmessage:
    performWork (yana 5ms)
    shouldYield → true
    port.postMessage(null)
    
... va h.k.
```

Har macrotask orasida browser paint qiladi. Agar input keldi yoki animation rAF'i bor — ular ham ishlaydi.

### `setTimeout(fn, 0)` nima uchun ishlatilmadi

`setTimeout(fn, 0)` ham macrotask rejalashtiradi. Lekin:

1. **Min delay** — HTML5 specification `setTimeout(fn, 0)` uchun minimum 4ms delay belgilab qo'ygan (nested case'larda)
2. **Throttling** — background tab'larda setTimeout 1000ms'gacha throttle qilinadi
3. **Less efficient** — har timeout har timer queue'ga qo'shiladi (memory)

`MessageChannel.postMessage()` — minimal delay (mikrosaniyalarda) va throttling yo'q.

<details>
<summary><strong>Under the Hood</strong></summary>

**Scheduler MessageChannel implementation:**

```typescript
// scheduler/src/forks/Scheduler.js  (R18-R19 manba; `SchedulerPostTask.js` alohida fork `scheduler.postTask()` API uchun)
let isMessageLoopRunning = false;
let scheduledHostCallback = null;
let taskTimeoutID = -1;

const channel = new MessageChannel();
const port = channel.port2;

channel.port1.onmessage = performWorkUntilDeadline;

function performWorkUntilDeadline() {
  if (scheduledHostCallback !== null) {
    const currentTime = getCurrentTime();
    
    // Frame budget reset
    startTime = currentTime;
    
    let hasMoreWork = true;
    try {
      hasMoreWork = scheduledHostCallback(true, currentTime);
    } finally {
      if (hasMoreWork) {
        schedulePerformWorkUntilDeadline();
      } else {
        isMessageLoopRunning = false;
        scheduledHostCallback = null;
      }
    }
  } else {
    isMessageLoopRunning = false;
  }
  
  needsPaint = false;
}

function schedulePerformWorkUntilDeadline() {
  if (!isMessageLoopRunning) {
    isMessageLoopRunning = true;
    port.postMessage(null);
  }
}

function requestHostCallback(callback) {
  scheduledHostCallback = callback;
  if (!isMessageLoopRunning) {
    isMessageLoopRunning = true;
    schedulePerformWorkUntilDeadline();
  }
}
```

**Server-side rendering (SSR) holati:**

`MessageChannel` browser'da asosiy yield mexanizmi, lekin scheduler avval `setImmediate`'ni tekshiradi. Node.js'da `setImmediate` mavjud, shuning uchun server'da o'sha ishlatiladi; MessageChannel ham, `setImmediate` ham bo'lmasa — `setTimeout(0)` (`scheduler/src/forks/Scheduler.js` ichida feature detection):

```typescript
// Soddalashtirilgan feature detection (real manba: `scheduler/src/forks/Scheduler.js`)
let schedulePerformWorkUntilDeadline;
if (typeof localSetImmediate === 'function') {
  // Node.js — MessageChannel yield qila olmaydi, setImmediate ishlatiladi
  schedulePerformWorkUntilDeadline = () => localSetImmediate(performWorkUntilDeadline);
} else if (typeof MessageChannel !== 'undefined') {
  // Browser — asosiy yo'l
  const channel = new MessageChannel();
  channel.port1.onmessage = performWorkUntilDeadline;
  schedulePerformWorkUntilDeadline = () => channel.port2.postMessage(null);
} else {
  // Oxirgi fallback
  schedulePerformWorkUntilDeadline = () => localSetTimeout(performWorkUntilDeadline, 0);
}
```

**SSR concurrent rendering:**
- `renderToString` — to'liq sync (Suspense streaming yo'q).
- `renderToReadableStream` (R18+ Web Streams) va `renderToPipeableStream` (R18+ Node) — Suspense bilan **concurrent** streaming SSR (boundary'lar bo'yicha incremental flush; client'dagi 5ms yield budget ishlatmaydi — Suspense boundary commit trigger qiladi).

**`scheduling.isInputPending()` API:**

```typescript
// React optional optimization (Chrome 87+)
function shouldYieldToHost() {
  const timeElapsed = getCurrentTime() - startTime;
  
  if (timeElapsed < frameYieldMs) {
    return false;
  }
  
  if (enableIsInputPending) {
    if (needsPaint) {
      return true;
    }
    
    if (timeElapsed < continuousYieldMs) {
      // Pending input bo'lsa — yield, yo'qsa — davom
      if (isInputPending !== null) {
        return isInputPending();
      }
      return false;
    }
    
    if (timeElapsed < maxYieldMs) {
      if (isInputPending !== null) {
        return isInputPending(continuousOptions);
      }
      return false;
    }
  }
  
  return true;
}
```

`navigator.scheduling.isInputPending()` browser API React'ga **input bor-yo'qligini** so'rashga imkon beradi. Input yo'q bo'lsa — budget'ni `maxYieldMs = 300` ms gacha cho'zish mumkin (`continuousYieldMs = 50` ms oraliq threshold; kerakmas yield kamaytirish).

Bu API faqat Chrome'da bor (Safari, Firefox'da yo'q). Boshqa browser'larda standart 5ms budget.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`requestIdleCallback` vs `MessageChannel` taqqoslash:

```typescript
// requestIdleCallback (eski yondashuv)
function rICWork(deadline) {
  while (deadline.timeRemaining() > 0 && hasWork()) {
    doWork();
  }
  
  if (hasWork()) {
    requestIdleCallback(rICWork);
  }
}

requestIdleCallback(rICWork);
// Frequency: faqat idle period'da (past, deterministic emas)
// Cross-browser: yo'q (Safari)

// MessageChannel (React'ning yondashuvi)
const channel = new MessageChannel();
let hasMore = true;

channel.port1.onmessage = () => {
  const start = performance.now();
  
  while (hasMore && performance.now() - start < 5) {
    hasMore = doWork();
  }
  
  if (hasMore) {
    channel.port2.postMessage(null);
  }
};

channel.port2.postMessage(null);
// Frequency: har macrotask (har frame'da bir nechta)
// Cross-browser: barcha modern browserlar
```

Polyfill misol (mental — React buni ichkidan qiladi):

```typescript
// React Scheduler — minimal polyfill version
const scheduler = {
  port: null as MessagePort | null,
  callback: null as Function | null,
  
  init() {
    const channel = new MessageChannel();
    channel.port1.onmessage = () => {
      if (this.callback) {
        const cb = this.callback;
        this.callback = null;
        cb();
      }
    };
    this.port = channel.port2;
  },
  
  schedule(fn: Function) {
    this.callback = fn;
    this.port?.postMessage(null);
  },
};

scheduler.init();

// Ishlatish
scheduler.schedule(() => {
  console.log('Yangi macrotask');
});
```

Inspect React Scheduler timing:

```tsx
function PerfMonitor() {
  useEffect(() => {
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.entryType === 'measure') {
          console.log(`${entry.name}: ${entry.duration.toFixed(2)}ms`);
        }
      }
    });
    
    observer.observe({ entryTypes: ['measure'] });
    return () => observer.disconnect();
  }, []);

  // React internal'da Scheduler measurements
  // performance.measure('⚛️ React work', 'react-start', 'react-end');
  
  return null;
}

// React DevTools Profiler bu measurements'dan ishlaydi
```

</details>

---

## Edge Cases va Gotchas

### Lane combinatsiyalari — bir Fiber'da bir nechta priority

```tsx
function App() {
  const [count, setCount] = useState(0);
  const [, startTransition] = useTransition();

  function handleMixed() {
    setCount(c => c + 1);  // SyncLane
    startTransition(() => {
      setCount(c => c + 1);  // TransitionLane
    });
  }
}

// Counter Fiber.lanes = SyncLane | TransitionLane
// 2 ta render bo'ladi:
// 1. Sync render (sync updater) → count=1, commit
// 2. Transition render (transition updater) → count=2, commit
// Foydalanuvchi: 0 → 1 → 2 ko'radi
```

---

### Expiration nojaiz UX

```tsx
function HighFrequencyApp() {
  const [data, setData] = useState({});
  const [, startTransition] = useTransition();

  // Har 50ms DefaultLane update (setInterval — React event handler emas)
  useEffect(() => {
    const id = setInterval(() => {
      setData(d => ({ ...d, tick: Date.now() }));
    }, 50);
    return () => clearInterval(id);
  }, []);

  function loadHeavy() {
    startTransition(() => {
      setData(d => ({ ...d, heavy: heavyComputation() }));
    });
  }
}

// Heavy transition har safar restart qilinadi (50ms da yangi DefaultLane update,
// u TransitionLane'dan yuqori priority)
// 5 sekund o'tgach — expired → sync sifatida bajariladi
// 5 sekund foydalanuvchi kutadi (transition commit bo'lmagan)
//
// To'g'rilash: DefaultLane update'larni kamaytirish, yoki heavy computation'ni splitting
```

---

### `flushSync` va Lane systemsi

```tsx
import { flushSync } from 'react-dom';

function handleClick() {
  flushSync(() => {
    setCount(c => c + 1);  // SyncLane bo'ladi
  });
  // Bu yerga yetganda DOM allaqachon yangilangan
  
  startTransition(() => {
    setData(d => ({...d, value: 1}));  // TransitionLane
  });
}

// flushSync — SyncLane bilan render majburan sync
// Bu — concurrent benefits'ni yo'qotadi
// Faqat third-party DOM library bilan integration uchun
```

---

### `useDeferredValue` — implicit transition

```tsx
function Search() {
  const [query, setQuery] = useState('');
  const deferred = useDeferredValue(query);
  
  // deferred — implicit transition lane bilan yangilanadi
  // query — sync (input darhol updated)
  // deferred — transition (kechiktirilgan)
  
  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <ExpensiveResult query={deferred} />
    </>
  );
}

// useDeferredValue — startTransition bilan birga (implicit)
// Lekin alohida lane ishlatadi (TransitionLane)
```

---

### Scheduler priority API — `unstable_*` ishlatmaslik

```tsx
// ❌ App kodda direct scheduler usage
import { unstable_scheduleCallback, unstable_NormalPriority } from 'scheduler';

unstable_scheduleCallback(unstable_NormalPriority, () => {
  // Work
});

// Sabab: unstable_ — public emas, React internal'lar uchun
// React versiya yangilanganda API o'zgarishi mumkin
// Breaking changes warning bermaydi (unstable bo'lgani uchun)

// ✅ React'ning rasmiy API'lari
function App() {
  const [, startTransition] = useTransition();
  
  startTransition(() => {
    // Work — TransitionLane priority
  });
}
```

---

## Common Mistakes

### ❌ Xato 1: `useTransition` har joyda

```tsx
// ❌ Har setState'ni transition'da o'rab olish
function App() {
  const [, startTransition] = useTransition();
  
  function handleClick() {
    startTransition(() => {
      setCount(c => c + 1);  // ❌ Click — sync bo'lishi kerak
    });
  }
}

// useTransition — non-urgent update'lar uchun
// Sync update (click, type) — direct setState ishlatish kerak
// Transition click handler'da — UI kechikadi (foydalanuvchi click javobini darhol kutmoqda)
```

```tsx
// ✅ To'g'ri — faqat heavy/non-urgent uchun
function App() {
  const [, startTransition] = useTransition();
  
  function handleClick() {
    setCount(c => c + 1);  // Sync — darhol javob
  }
  
  function handleSearch(q: string) {
    setQuery(q);  // Sync — input darhol updated
    
    startTransition(() => {
      setHeavyResults(filter(q));  // Transition — non-urgent
    });
  }
}
```

---

### ❌ Xato 2: Transition ichida side effect

```tsx
function Bad() {
  const [, startTransition] = useTransition();

  function handleClick() {
    startTransition(() => {
      setSearchResults([]);
      
      // ❌ Side effect startTransition callback ichida
      analytics.track('search', query);
      fetch('/api/log');
    });
  }
}

// startTransition callback'i SINXRON ishlaydi — side effect darhol, render
// boshlanmasdan oldin bajariladi. Bu side effect transition render commit
// bo'lishidan mustaqil:
// - Transition keyin yangi update bilan supersede bo'lsa ham track/fetch
//   allaqachon yuborilgan (commit bo'lmagan natija uchun ham network request)
// - "Transition muvaffaqiyatli yakunlandi" hodisasiga bog'lab bo'lmaydi
```

```tsx
// ✅ Side effect — useEffect ichida
function Good() {
  const [results, setResults] = useState([]);
  const [, startTransition] = useTransition();

  function handleClick() {
    startTransition(() => {
      setResults(search());
    });
  }

  useEffect(() => {
    if (results.length > 0) {
      analytics.track('search-results', results.length);
    }
  }, [results]);
}
```

---

### ❌ Xato 3: Heavy work transition ichida — har restart qaytariladi

```tsx
function Bad() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [, startTransition] = useTransition();

  function handleChange(value: string) {
    setQuery(value);
    
    startTransition(() => {
      // ❌ Har keypress'da heavy work
      // Restart bo'lganda yana qaytariladi
      const heavyResult = expensiveComputation(value);
      setResults(heavyResult);
    });
  }
}
```

```tsx
// ✅ useDeferredValue + useMemo — caching
function Good() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  
  // useMemo cache — bir xil deferredQuery uchun bir marta
  const results = useMemo(
    () => expensiveComputation(deferredQuery),
    [deferredQuery]
  );

  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <Results items={results} />
    </>
  );
}
```

---

### ❌ Xato 4: Scheduler priority'larni manual'da boshqarish

```tsx
// ❌ Direct scheduler API
import { unstable_runWithPriority, unstable_LowPriority } from 'scheduler';

function handleClick() {
  unstable_runWithPriority(unstable_LowPriority, () => {
    setCount(c => c + 1);  // Low priority bilan
  });
}
// Bu API unstable, app kodda ishlatish noto'g'ri
```

```tsx
// ✅ React'ning rasmiy API'i
function handleClick() {
  startTransition(() => {  // TransitionLane (low priority bilan birga)
    setCount(c => c + 1);
  });
}
```

---

### ❌ Xato 5: Concurrent invariants buzish

```tsx
function Bad() {
  let renderCount = 0;  // Tashqi mutable state
  
  function Counter() {
    renderCount++;  // ❌ Render davomida side effect
    
    return <p>Renders: {renderCount}</p>;
  }
}

// Concurrent rendering:
// - Render restart bo'lganda renderCount yana oshiriladi
// - Wasted work tufayli son haqiqiy emas
// - Strict Mode 2x render — son ikki barobar
```

```tsx
// ✅ useRef bilan render counting (lekin bu ham ehtiyotkorlik talab qiladi)
function Counter() {
  const renderCountRef = useRef(0);
  
  useEffect(() => {
    renderCountRef.current += 1;  // Effect — commit phase, atomic
  });

  return <p>Renders: {renderCountRef.current}</p>;
}
```

---

## Amaliy Mashqlar

### Mashq 1: Lane priority aniqlash (Oson)

Quyidagi har holat uchun qaysi lane ishlatilishini ayting:

```tsx
function App() {
  const [count, setCount] = useState(0);
  const [, startTransition] = useTransition();
  const deferred = useDeferredValue(count);

  function handleClick() {
    setCount(c => c + 1);  // ?
  }

  function handleScroll() {
    setCount(c => c + 1);  // ?
  }

  function handleSearch() {
    startTransition(() => {
      setCount(c => c + 1);  // ?
    });
  }

  useEffect(() => {
    const id = setInterval(() => {
      setCount(c => c + 1);  // ?
    }, 1000);
    return () => clearInterval(id);
  }, []);
}
```

<details>
<summary><strong>Javob</strong></summary>

| Joy | Lane | Sabab |
|-----|------|-------|
| `handleClick` | `SyncLane` | Click event handler — SyncLane bilan rejalashtiriladi |
| `handleScroll` | `InputContinuousLane` | Continuous input event (scroll, drag, mousemove) |
| `handleSearch` (transition) | `TransitionLane` | `useTransition` ichidagi setState |
| `useEffect` interval | `DefaultLane` | Default — event listener'siz, useEffect ichida |
| `deferred` (useDeferredValue) | `TransitionLane` | useDeferredValue — implicit transition |

</details>

---

### Mashq 2: Render restart sabab (O'rta)

Quyidagi senaryo'da nechta render boshlanadi va nechta commit bo'ladi?

```tsx
function App() {
  const [text, setText] = useState('');
  const [, startTransition] = useTransition();

  function handleType(value: string) {
    setText(value);  // SyncLane
    startTransition(() => {
      setExpensiveData(processData(value));  // TransitionLane
    });
  }
}

// Foydalanuvchi tezda yozadi: "a", "ab", "abc"
// Har keypress orasida ~30ms (transition render'i ulgurmaydi)
```

<details>
<summary><strong>Javob</strong></summary>

**Renderlar boshlanishi (taxminiy):**

```
"a" yozildi:
  - SyncLane render (text = "a") → commit (instant)
  - TransitionLane render boshlandi (expensiveData = processData("a"))
  - 30ms o'tdi, render davom etayapti

"ab" yozildi:
  - SyncLane render (text = "ab") → commit (instant)
  - Old TransitionLane render TASHLANDI (1 wasted)
  - Yangi TransitionLane render boshlandi (expensiveData = processData("ab"))
  - 30ms o'tdi

"abc" yozildi:
  - SyncLane render (text = "abc") → commit
  - Old TransitionLane render TASHLANDI (2 wasted)
  - Yangi TransitionLane render boshlandi
  - Foydalanuvchi to'xtagan
  - Render tugadi → commit
```

**Hisob:**
- Sync render boshlanishlari: 3 (har keypress)
- Sync commit'lar: 3
- Transition render boshlanishlari: 3 (har keypress yangi boshlash)
- Transition commit'lar: 1 (faqat oxirgi muvaffaqiyatli)
- Wasted work: 2 ta tashlangan transition render

**Optimization:** `useDeferredValue` + `useMemo` bilan heavy computation cache qilish — har transition render'da `processData` qaytarmasdan, deferredQuery uchun `useMemo` cache.

</details>

---

### Mashq 3: Starvation simulatsiyasi (Qiyin)

Quyidagi kod qachon transition commit bo'ladi?

```tsx
function App() {
  const [, startTransition] = useTransition();
  const [counter, setCounter] = useState(0);
  const [transitionState, setTransitionState] = useState('initial');

  // DefaultLane update har 100ms (setInterval — React event handler emas)
  useEffect(() => {
    const id = setInterval(() => {
      setCounter(c => c + 1);  // DefaultLane
    }, 100);
    return () => clearInterval(id);
  }, []);

  // Transition update bir marta
  useEffect(() => {
    startTransition(() => {
      setTransitionState('updated');  // TransitionLane
    });
  }, []);

  return <div>{counter} - {transitionState}</div>;
}
```

<details>
<summary><strong>Javob</strong></summary>

**Scenario:**

1. Mount: counter=0, transitionState="initial"
2. useEffect interval boshlanadi (sync update har 100ms)
3. useEffect transition: startTransition → setTransitionState

**Render cycle:**

```
T=0: Mount commit (counter=0, transitionState="initial")
T=0: Transition setState → TransitionLane register (expirationTime = T+5000)
T=0: TransitionLane render boshlanadi
T=100ms: DefaultLane update — counter=1
  - DefaultLane render → commit (counter=1, transitionState="initial")
  - Transition render TASHLANADI (restart)
  - Yangi transition render boshlanadi
T=200ms: DefaultLane update — counter=2
  - DefaultLane render → commit (counter=2)
  - Transition render TASHLANADI yana
T=300ms: counter=3, transition tashlanadi
T=400ms: counter=4, transition tashlanadi
...
T=4900ms: counter=49, transition tashlanadi
T=5000ms: TransitionLane EXPIRED!
T=5000ms: DefaultLane update — counter=50
  - Render boshlanadi
  - Lekin TransitionLane ham expired — sync sifatida ishlanadi
  - Render: counter=50, transitionState="updated"
  - Commit
```

**Javob:** Transition **5 sekund kechikishdan keyin** commit bo'ladi (expiration tufayli). Foydalanuvchi 5 sekund "initial" matnni ko'radi, keyin "updated" ga o'zgaradi.

Bu — starvation prevention'ning real ishlashi. Agar prevention bo'lmasa, transition hech qachon commit bo'lmasdi.

</details>

---

### Mashq 4: useDeferredValue vs useTransition (Qiyin)

Quyidagi ikki kod nimasi bilan farq qiladi?

```tsx
// Variant A: useTransition
function VariantA() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<Item[]>([]);
  const [, startTransition] = useTransition();

  function handleChange(value: string) {
    setQuery(value);
    startTransition(() => {
      setResults(search(value));
    });
  }
}

// Variant B: useDeferredValue
function VariantB() {
  const [query, setQuery] = useState('');
  const deferred = useDeferredValue(query);
  
  const results = useMemo(() => search(deferred), [deferred]);

  function handleChange(value: string) {
    setQuery(value);
  }
}
```

<details>
<summary><strong>Javob</strong></summary>

**Asosiy farqlar:**

| Aspect | Variant A (`useTransition`) | Variant B (`useDeferredValue`) |
|--------|----------------------------|-------------------------------|
| **State controllability** | Ikkita state (`query` va `results`) | Bitta state (`query`), `deferred` derived |
| **Side effects** | `setResults` direct chaqiriladi | `useMemo` natijasi (no setState) |
| **Wasted work** | Har transition restart'da `search` qayta chaqiriladi | `useMemo` cache (deferred bir xil bo'lsa skip) |
| **Code complexity** | Ikki state, explicit transition | Bitta state, automatic deferral |
| **Stale value indicator** | `isPending` bool | `query !== deferred` taqqoslash |

**Variant A foydasi:**
- Explicit control — qaysi update transition'da bo'lishini siz aytasiz
- `isPending` darhol mavjud (loading indicator)
- Birdan bir nechta state'larni transition qilish mumkin

**Variant B foydasi:**
- Memoization built-in (useMemo bilan tabiiy)
- Wasted work kamroq (deferred reuse, search qayta chaqirilmaydi)
- Soddroq mental model — query → deferred → results
- `useDeferredValue(externalProp)` — tashqi prop'ni defer qilish (useTransition bilan qiyin)

**Qachon qaysi:**

- **`useTransition`** — siz transition chegarasini kontrol qila olganingizda (event handler ichida)
- **`useDeferredValue`** — qiymat tashqaridan keladi (prop) yoki kontrol qila olmaysiz, lekin defer qilish kerak

Bu mavzu `22-concurrent-hooks.md` da chuqurroq yoritiladi.

</details>

---

### Mashq 5: Scheduler architecture tahlili (Qiyin)

Nima uchun React Scheduler **alohida npm paket** sifatida ajratilgan? 4 ta sabab keltiring.

<details>
<summary><strong>Javob</strong></summary>

**1. Separation of Concerns:**

Scheduler React'ga maxsus emas — bu **umumiy purpose** task scheduling library. React bu paketdan foydalanadi, lekin scheduler React'siz ham ishlay oladi (Reach UI, Recoil, va boshqa library'lar potential consumer'lar).

**2. Independent Versioning:**

Scheduler API barqaror — React Reconciler ko'p o'zgarishlar qilsa-da, scheduler interface'i stable qoladi. R16+ versiyalar barcha bir xil scheduler bilan ishlaydi. Yangi React versiyasi yangi scheduler talab qilmaydi (semver).

**3. Custom Renderer'lar:**

`react-reconciler` ishlatuvchi custom renderer'lar (Ink, react-three-fiber, va h.k.) ham scheduler'dan foydalanadi. Agar scheduler React'ning ichida bo'lsa, har custom renderer scheduler'ni qayta yozishi kerak edi.

**4. Test Isolation:**

Scheduler unit test'lar React'siz yoziladi. Scheduler internal'lari (priority queue, time slicing) alohida test qilinadi. Test mocks (`scheduler/unstable_mock`) React tests'da scheduler'ni mock qilishga imkon beradi.

**Bonus 5: Ecosystem Reusability:**

Scheduler API npm'da public — boshqa library'lar `unstable_scheduleCallback` ishlatishi mumkin (lekin unstable_ prefix'i sababli risky). Misol: Recoil scheduler bilan integration qiladi (concurrent rendering bilan compatibility).

</details>

---

## Xulosa

Bu bo'limda Scheduler va Lanes mexanizmi yoritildi:

- **Concurrent Rendering** — render bo'lib-bo'lib bajariladi, har 5ms'da yield
- **Scheduler package** — alohida `scheduler/` npm, priority-based task scheduling
- **Priority levels** — Immediate, UserBlocking, Normal, Low, Idle
- **Lanes model (R18+)** — 31-bit bitmap, bir Fiber'da bir nechta priority
- **Lane types** — SyncLane, InputContinuousLane, DefaultLane, TransitionLane (16), IdleLane
- **Time slicing** — `frameYieldMs = 5`, har 5ms'da browser'ga yo'l berish
- **Interruptible rendering** — yangi update kelganda eski render TASHLANADI
- **Expiration** — starvation prevention, lane uzoq kutgach sync sifatida bajariladi
- **MessageChannel** — `requestIdleCallback`'dan ko'ra deterministic va cross-browser

Bu mexanizm — `useTransition`, `useDeferredValue`, `<Suspense>` kabi modern React API'larining asosi.

**QISM 2 (REACT INTERNALS) tugadi** — Fiber, Reconciliation, Scheduler/Lanes barchasi yoritildi. Keyingi bo'lim — Hydration — server HTML'ni client React'ga ulash mexanikasini yoritadi.

---

**Keyingi bo'lim:** [06-hydration.md](06-hydration.md) — SSR + Hydration concept, `hydrateRoot` vs `createRoot`, hydration mismatch sabablari (Date.now/Math.random/browser-only API/conditional env), `suppressHydrationWarning`, R18 selective hydration (Suspense bilan), R18 streaming hydration, R19 hydration improvements (better error messages, `onRecoverableError`, mismatch diffing).
