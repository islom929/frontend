# Vue Coding Challenges — Interview

> **30 ta coding challenge** — Composables (useEventListener, useDebounceFn/useThrottleFn, useIntersectionObserver, useElementSize, useMediaQuery, useLocalStorage, useFetch, useOnClickOutside, useWindowSize, useClipboard, usePrevious, useToggle), Custom Directives (v-click-outside, v-tooltip, v-focus-trap, v-lazy-load, v-permission, v-longpress), Toast Plugin, Provide/Inject DI, Modal Component, Async + Suspense + Error Boundary, Render Function (Dynamic Heading), **Mini Reactivity System**, TransitionGroup FLIP, defineCustomElement, Multi v-model, Custom Renderer (Canvas), Infinite Scroll, Virtual List.

**Daraja taqsimoti:** 2 [Junior+] · 13 [Middle] · 10 [Middle+] · 5 [Senior]

---

## Mundarija

1. [`useEventListener` — auto cleanup](#challenge-1-useeventlistener--auto-cleanup-middle)
2. [`useDebounceFn` / `useThrottleFn`](#challenge-2-usedebouncefn--usethrottlefn-middle)
3. [`useIntersectionObserver`](#challenge-3-useintersectionobserver-middle)
4. [`useElementSize` (ResizeObserver)](#challenge-4-useelementsize-resizeobserver-middle)
5. [`useMediaQuery`](#challenge-5-usemediaquery-middle)
6. [`v-click-outside` directive](#challenge-6-v-click-outside-directive-middle)
7. [`v-tooltip` directive](#challenge-7-v-tooltip-directive-middle)
8. [`v-focus-trap` directive](#challenge-8-v-focus-trap-directive-middle)
9. [Toast notification plugin](#challenge-9-toast-notification-plugin-middle)
10. [Provide/Inject DI pattern](#challenge-10-provideinject-di-pattern-middle)
11. [Modal komponent (Teleport + defineModel + Transition)](#challenge-11-modal-komponent-teleport--definemodel--transition-middle)
12. [Async component + Suspense + Error Boundary](#challenge-12-async-component--suspense--error-boundary-middle)
13. [Render function dynamic heading](#challenge-13-render-function-dynamic-heading-senior)
14. [Mini reactivity system](#challenge-14-mini-reactivity-system-senior)
15. [TransitionGroup FLIP animation](#challenge-15-transitiongroup-flip-animation-middle)
16. [`defineCustomElement` Web Component](#challenge-16-definecustomelement-web-component-senior)
17. [`defineModel` multi v-model komponent](#challenge-17-definemodel-multi-v-model-komponent-middle)
18. [Custom renderer (Canvas)](#challenge-18-custom-renderer-canvas-senior)
19. [`useToggle` — boolean state management](#challenge-19-usetoggle--boolean-state-management-junior)
20. [`useLocalStorage<T>` — SSR-safe typed storage](#challenge-20-uselocalstorage--ssr-safe-typed-storage-middle)
21. [`useFetch<T>` — abort + loading + error + refetch](#challenge-21-usefetch--abort--loading--error--refetch-middle)
22. [`useOnClickOutside` — multiple elements](#challenge-22-useonclickoutside--multiple-elements-middle)
23. [`useWindowSize` — SSR-safe + debounce](#challenge-23-usewindowsize--ssr-safe--debounce-middle)
24. [`useClipboard` — Clipboard API + fallback](#challenge-24-useclipboard--clipboard-api--fallback-middle)
25. [`usePrevious<T>` — oldingi qiymat](#challenge-25-useprevious--oldingi-qiymat-junior)
26. [`v-lazy-load` — IntersectionObserver image loading](#challenge-26-v-lazy-load--intersectionobserver-image-loading-middle)
27. [`v-permission` — role-based visibility](#challenge-27-v-permission--role-based-visibility-middle)
28. [`v-longpress` — long press event](#challenge-28-v-longpress--long-press-event-middle)
29. [Infinite scroll component](#challenge-29-infinite-scroll-component-middle)
30. [Virtual list component (fixed height)](#challenge-30-virtual-list-component-fixed-height-senior)

---

## Challenge 1: `useEventListener` — auto cleanup [Middle]

**Talab:** Composable yarating: `useEventListener(target, event, handler)` — `addEventListener` + `removeEventListener` auto cleanup `onBeforeUnmount`'da.

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useEventListener.ts
import { onMounted, onBeforeUnmount, watch, unref, type Ref } from 'vue'

type MaybeRef<T> = T | Ref<T>

export function useEventListener<K extends keyof WindowEventMap>(
  target: MaybeRef<EventTarget | null | undefined>,
  event: K,
  handler: (e: WindowEventMap[K]) => void,
  options?: AddEventListenerOptions,
): void
export function useEventListener(
  target: MaybeRef<EventTarget | null | undefined>,
  event: string,
  handler: EventListener,
  options?: AddEventListenerOptions,
): void

export function useEventListener(
  target: MaybeRef<EventTarget | null | undefined>,
  event: string,
  handler: EventListener,
  options?: AddEventListenerOptions,
) {
  let cleanup = () => {}

  function attach(el: EventTarget) {
    el.addEventListener(event, handler, options)
    cleanup = () => el.removeEventListener(event, handler, options)
  }

  onMounted(() => {
    const el = unref(target)
    if (el) attach(el)
  })

  // Re-attach if target changes
  watch(() => unref(target), (newEl, oldEl) => {
    cleanup()
    if (newEl) attach(newEl)
  })

  onBeforeUnmount(() => cleanup())
}
```

Usage:

```vue
<script setup lang="ts">
import { ref, useTemplateRef } from 'vue'
import { useEventListener } from '@/composables/useEventListener'

const buttonRef = useTemplateRef<HTMLButtonElement>('btn')

useEventListener(window, 'resize', () => {
  console.log('Window resized')
})

useEventListener(buttonRef, 'click', () => {
  console.log('Button clicked')
})
</script>

<template>
  <button ref="btn">Click me</button>
</template>
```

</details>

---

## Challenge 2: `useDebounceFn` / `useThrottleFn` [Middle]

**Talab:** Composable: `useDebounceFn(fn, delay)` — function call'lar debounce qiladi. `useThrottleFn(fn, delay)` — throttle.

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useDebounceFn.ts
import { onBeforeUnmount } from 'vue'

export function useDebounceFn<T extends (...args: any[]) => any>(
  fn: T,
  delay: number = 300,
): (...args: Parameters<T>) => void {
  let timeoutId: ReturnType<typeof setTimeout> | null = null

  const debounced = (...args: Parameters<T>) => {
    if (timeoutId) clearTimeout(timeoutId)
    timeoutId = setTimeout(() => fn(...args), delay)
  }

  onBeforeUnmount(() => {
    if (timeoutId) clearTimeout(timeoutId)
  })

  return debounced
}
```

```typescript
// composables/useThrottleFn.ts
import { onBeforeUnmount } from 'vue'

export function useThrottleFn<T extends (...args: any[]) => any>(
  fn: T,
  delay: number = 300,
): (...args: Parameters<T>) => void {
  let lastCall = 0
  let timeoutId: ReturnType<typeof setTimeout> | null = null

  const throttled = (...args: Parameters<T>) => {
    const now = Date.now()
    const remaining = delay - (now - lastCall)

    if (remaining <= 0) {
      lastCall = now
      fn(...args)
    } else if (!timeoutId) {
      timeoutId = setTimeout(() => {
        lastCall = Date.now()
        timeoutId = null
        fn(...args)
      }, remaining)
    }
  }

  onBeforeUnmount(() => {
    if (timeoutId) clearTimeout(timeoutId)
  })

  return throttled
}
```

Usage:

```vue
<script setup lang="ts">
import { ref } from 'vue'
import { useDebounceFn, useThrottleFn } from '@/composables'
import { useEventListener } from '@/composables/useEventListener'

const search = ref('')

const debouncedSearch = useDebounceFn(async (q: string) => {
  console.log('Searching:', q)
  const res = await fetch(`/api/search?q=${q}`)
  // ...
}, 500)

const throttledScroll = useThrottleFn(() => {
  console.log('Scroll:', window.scrollY)
}, 100)

// useEventListener auto cleanup qiladi
useEventListener(window, 'scroll', throttledScroll)
</script>

<template>
  <input v-model="search" @input="debouncedSearch(search)" />
</template>
```

</details>

---

## Challenge 3: `useIntersectionObserver` [Middle]

**Talab:** Composable: target element viewport'ga kirsa/chiqsa reactive `isIntersecting` qaytaradi.

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useIntersectionObserver.ts
import { ref, onMounted, onBeforeUnmount, watch, unref, type Ref } from 'vue'

interface UseIntersectionObserverOptions extends IntersectionObserverInit {
  immediate?: boolean
}

export function useIntersectionObserver(
  target: Ref<HTMLElement | null>,
  options: UseIntersectionObserverOptions = {},
) {
  const isIntersecting = ref(false)
  const intersectionRatio = ref(0)
  let observer: IntersectionObserver | null = null

  function setup(el: HTMLElement) {
    observer = new IntersectionObserver(([entry]) => {
      isIntersecting.value = entry.isIntersecting
      intersectionRatio.value = entry.intersectionRatio
    }, {
      root: options.root,
      rootMargin: options.rootMargin,
      threshold: options.threshold,
    })

    observer.observe(el)
  }

  function cleanup() {
    observer?.disconnect()
    observer = null
  }

  onMounted(() => {
    const el = unref(target)
    if (el) setup(el)
  })

  watch(() => unref(target), (newEl, oldEl) => {
    cleanup()
    if (newEl) setup(newEl)
  })

  onBeforeUnmount(cleanup)

  return {
    isIntersecting,
    intersectionRatio,
    stop: cleanup,
  }
}
```

Usage:

```vue
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import { useIntersectionObserver } from '@/composables/useIntersectionObserver'

const targetRef = useTemplateRef<HTMLDivElement>('target')

const { isIntersecting } = useIntersectionObserver(targetRef, {
  rootMargin: '100px',
  threshold: 0.5,
})
</script>

<template>
  <div style="height: 200vh">
    <div ref="target" class="target">
      {{ isIntersecting ? 'Visible!' : 'Not visible' }}
    </div>
  </div>
</template>
```

</details>

---

## Challenge 4: `useElementSize` (ResizeObserver) [Middle]

**Talab:** Composable: element size'ni track qiladi (`width`, `height`). ResizeObserver ishlatib.

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useElementSize.ts
import { ref, onMounted, onBeforeUnmount, watch, unref, type Ref } from 'vue'

export function useElementSize(target: Ref<HTMLElement | null>) {
  const width = ref(0)
  const height = ref(0)

  let observer: ResizeObserver | null = null

  function setup(el: HTMLElement) {
    observer = new ResizeObserver(([entry]) => {
      const { width: w, height: h } = entry.contentRect
      width.value = w
      height.value = h
    })

    observer.observe(el)
  }

  function cleanup() {
    observer?.disconnect()
    observer = null
  }

  onMounted(() => {
    const el = unref(target)
    if (el) {
      width.value = el.offsetWidth
      height.value = el.offsetHeight
      setup(el)
    }
  })

  watch(() => unref(target), (newEl) => {
    cleanup()
    if (newEl) setup(newEl)
  })

  onBeforeUnmount(cleanup)

  return { width, height }
}
```

Usage:

```vue
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import { useElementSize } from '@/composables/useElementSize'

const boxRef = useTemplateRef<HTMLDivElement>('box')
const { width, height } = useElementSize(boxRef)
</script>

<template>
  <div ref="box" class="resizable">
    <p>Size: {{ width.toFixed(0) }} x {{ height.toFixed(0) }}</p>
  </div>
</template>

<style>
.resizable { resize: both; overflow: auto; min-width: 100px; min-height: 100px; border: 1px solid; }
</style>
```

</details>

---

## Challenge 5: `useMediaQuery` [Middle]

**Talab:** Composable: media query string'ni reactive boolean qaytaradi.

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useMediaQuery.ts
import { ref, onMounted, onBeforeUnmount } from 'vue'

export function useMediaQuery(query: string) {
  const matches = ref(false)
  let mediaQuery: MediaQueryList | null = null

  function update(e: MediaQueryListEvent | MediaQueryList) {
    matches.value = e.matches
  }

  onMounted(() => {
    mediaQuery = window.matchMedia(query)
    matches.value = mediaQuery.matches
    mediaQuery.addEventListener('change', update)
  })

  onBeforeUnmount(() => {
    mediaQuery?.removeEventListener('change', update)
  })

  return matches
}
```

Usage:

```vue
<script setup lang="ts">
import { useMediaQuery } from '@/composables/useMediaQuery'

const isMobile = useMediaQuery('(max-width: 768px)')
const isDarkMode = useMediaQuery('(prefers-color-scheme: dark)')
const isLandscape = useMediaQuery('(orientation: landscape)')
</script>

<template>
  <div :class="{ mobile: isMobile, dark: isDarkMode, landscape: isLandscape }">
    <p>Mobile: {{ isMobile }}</p>
    <p>Dark mode: {{ isDarkMode }}</p>
    <p>Landscape: {{ isLandscape }}</p>
  </div>
</template>
```

</details>

---

## Challenge 6: `v-click-outside` directive [Middle]

**Talab:** Custom directive: element tashqarisida click bo'lsa handler chaqirsin.

<details>
<summary><strong>Javob</strong></summary>

```typescript
// directives/vClickOutside.ts
import type { Directive } from 'vue'

interface ClickOutsideEl extends HTMLElement {
  _clickOutsideHandler?: (e: MouseEvent) => void
}

export const vClickOutside: Directive<ClickOutsideEl, (e: MouseEvent) => void> = {
  mounted(el, binding) {
    if (typeof binding.value !== 'function') {
      console.warn('v-click-outside expects a function')
      return
    }

    el._clickOutsideHandler = (e: MouseEvent) => {
      if (!el.contains(e.target as Node)) {
        binding.value(e)
      }
    }

    // Defer to avoid same-cycle trigger
    setTimeout(() => {
      if (el._clickOutsideHandler) {
        document.addEventListener('click', el._clickOutsideHandler)
      }
    }, 0)
  },

  beforeUnmount(el) {
    if (el._clickOutsideHandler) {
      document.removeEventListener('click', el._clickOutsideHandler)
      delete el._clickOutsideHandler
    }
  },
}
```

Register globally:

```typescript
// main.ts
import { createApp } from 'vue'
import { vClickOutside } from '@/directives/vClickOutside'

const app = createApp(App)
app.directive('click-outside', vClickOutside)
```

Usage (dropdown menu):

```vue
<script setup lang="ts">
import { ref } from 'vue'

const isOpen = ref(false)
const items = ['Profile', 'Settings', 'Logout']
</script>

<template>
  <div class="dropdown" v-click-outside="() => isOpen = false">
    <button @click="isOpen = !isOpen">Menu</button>
    <ul v-if="isOpen" class="menu">
      <li v-for="item in items" :key="item">{{ item }}</li>
    </ul>
  </div>
</template>

<style scoped>
.dropdown { position: relative; }
.menu {
  position: absolute;
  top: 100%;
  background: white;
  border: 1px solid #ccc;
  list-style: none;
  padding: 0.5rem 0;
}
.menu li { padding: 0.5rem 1rem; cursor: pointer; }
.menu li:hover { background: #f0f0f0; }
</style>
```

</details>

---

## Challenge 7: `v-tooltip` directive [Middle+]

**Talab:** Custom directive: element'ga hover bo'lganda tooltip ko'rsatsin. `v-tooltip="'text'"` syntax.

<details>
<summary><strong>Javob</strong></summary>

```typescript
// directives/vTooltip.ts
import type { Directive, DirectiveBinding } from 'vue'

interface TooltipEl extends HTMLElement {
  _tooltipEl?: HTMLDivElement
  _tooltipHandlers?: {
    enter: () => void
    leave: () => void
  }
}

function positionTooltip(el: HTMLElement, tooltip: HTMLDivElement) {
  const rect = el.getBoundingClientRect()
  tooltip.style.top = `${rect.bottom + window.scrollY + 8}px`
  tooltip.style.left = `${rect.left + window.scrollX + rect.width / 2}px`
  tooltip.style.transform = 'translateX(-50%)'
}

export const vTooltip: Directive<TooltipEl, string> = {
  mounted(el, binding) {
    const tooltip = document.createElement('div')
    tooltip.className = 'app-tooltip'
    tooltip.textContent = binding.value
    tooltip.style.cssText = `
      position: absolute;
      background: rgba(0, 0, 0, 0.85);
      color: white;
      padding: 0.5rem 0.75rem;
      border-radius: 4px;
      font-size: 0.875rem;
      pointer-events: none;
      opacity: 0;
      transition: opacity 200ms;
      z-index: 9999;
    `
    document.body.appendChild(tooltip)

    el._tooltipEl = tooltip
    el._tooltipHandlers = {
      enter: () => {
        positionTooltip(el, tooltip)
        tooltip.style.opacity = '1'
      },
      leave: () => {
        tooltip.style.opacity = '0'
      },
    }

    el.addEventListener('mouseenter', el._tooltipHandlers.enter)
    el.addEventListener('mouseleave', el._tooltipHandlers.leave)
  },

  updated(el, binding) {
    if (el._tooltipEl && binding.value !== binding.oldValue) {
      el._tooltipEl.textContent = binding.value
    }
  },

  beforeUnmount(el) {
    if (el._tooltipHandlers) {
      el.removeEventListener('mouseenter', el._tooltipHandlers.enter)
      el.removeEventListener('mouseleave', el._tooltipHandlers.leave)
    }
    if (el._tooltipEl) {
      el._tooltipEl.remove()
    }
    delete el._tooltipEl
    delete el._tooltipHandlers
  },
}
```

Usage:

```vue
<template>
  <button v-tooltip="'Click to save'">Save</button>
  <a v-tooltip="'Opens in new window'" href="..." target="_blank">Link</a>
  <span v-tooltip="`Updated: ${date}`">{{ name }}</span>
</template>
```

</details>

---

## Challenge 8: `v-focus-trap` directive [Middle+]

**Talab:** Custom directive: element ichida focus'ni "trap" qilsin (Tab/Shift+Tab faqat ichkari element'lar orasida). Modal/dialog uchun.

<details>
<summary><strong>Javob</strong></summary>

```typescript
// directives/vFocusTrap.ts
import type { Directive } from 'vue'

interface FocusTrapEl extends HTMLElement {
  _focusTrapHandler?: (e: KeyboardEvent) => void
  _previousFocus?: HTMLElement | null
}

const FOCUSABLE_SELECTOR = [
  'a[href]',
  'button:not([disabled])',
  'input:not([disabled])',
  'select:not([disabled])',
  'textarea:not([disabled])',
  '[tabindex]:not([tabindex="-1"])',
].join(',')

function getFocusable(el: HTMLElement): HTMLElement[] {
  return Array.from(el.querySelectorAll<HTMLElement>(FOCUSABLE_SELECTOR))
}

export const vFocusTrap: Directive<FocusTrapEl, boolean | undefined> = {
  mounted(el, binding) {
    if (binding.value === false) return

    el._previousFocus = document.activeElement as HTMLElement

    el._focusTrapHandler = (e: KeyboardEvent) => {
      if (e.key !== 'Tab') return

      const focusable = getFocusable(el)
      if (focusable.length === 0) {
        e.preventDefault()
        return
      }

      const first = focusable[0]
      const last = focusable[focusable.length - 1]
      const active = document.activeElement as HTMLElement

      if (e.shiftKey) {
        if (active === first) {
          e.preventDefault()
          last.focus()
        }
      } else {
        if (active === last) {
          e.preventDefault()
          first.focus()
        }
      }
    }

    el.addEventListener('keydown', el._focusTrapHandler)

    // Focus first element on mount
    const focusable = getFocusable(el)
    if (focusable.length > 0) {
      focusable[0].focus()
    }
  },

  beforeUnmount(el) {
    if (el._focusTrapHandler) {
      el.removeEventListener('keydown', el._focusTrapHandler)
      delete el._focusTrapHandler
    }

    // Restore previous focus
    if (el._previousFocus && typeof el._previousFocus.focus === 'function') {
      el._previousFocus.focus()
    }
    delete el._previousFocus
  },
}
```

Usage (modal):

```vue
<script setup lang="ts">
import { ref } from 'vue'

const showModal = ref(false)
</script>

<template>
  <button @click="showModal = true">Open Modal</button>

  <div v-if="showModal" class="modal-backdrop">
    <div v-focus-trap class="modal" role="dialog">
      <h2>Confirm Action</h2>
      <input type="text" placeholder="Reason" />
      <textarea placeholder="Comments"></textarea>
      <button @click="showModal = false">Cancel</button>
      <button @click="showModal = false">Confirm</button>
    </div>
  </div>
</template>
```

Tab/Shift+Tab modal ichida cycle qiladi. Modal close paytida focus avvalgi joyga qaytadi.

</details>

---

## Challenge 9: Toast notification plugin [Middle+]

**Talab:** Plugin: `app.$toast('message', { type: 'success' })` chaqirilsa toast ko'rsatsin (auto-dismiss).

<details>
<summary><strong>Javob</strong></summary>

```typescript
// plugins/toast.ts
import { createApp, h, ref, defineComponent, type App } from 'vue'

interface Toast {
  id: number
  message: string
  type: 'info' | 'success' | 'warning' | 'error'
  duration: number
}

const toasts = ref<Toast[]>([])
let toastId = 0

const ToastContainer = defineComponent({
  setup() {
    return () => h(
      'div',
      {
        style: {
          position: 'fixed',
          top: '1rem',
          right: '1rem',
          zIndex: 9999,
          display: 'flex',
          flexDirection: 'column',
          gap: '0.5rem',
        }
      },
      toasts.value.map((toast) =>
        h('div', {
          key: toast.id,
          style: {
            padding: '1rem 1.5rem',
            borderRadius: '6px',
            color: 'white',
            minWidth: '250px',
            background: {
              info: '#3b82f6',
              success: '#10b981',
              warning: '#f59e0b',
              error: '#ef4444',
            }[toast.type],
            boxShadow: '0 4px 6px rgba(0,0,0,0.1)',
          }
        }, toast.message)
      )
    )
  }
})

function showToast(
  message: string,
  options: { type?: Toast['type']; duration?: number } = {},
) {
  const toast: Toast = {
    id: ++toastId,
    message,
    type: options.type ?? 'info',
    duration: options.duration ?? 3000,
  }

  toasts.value.push(toast)

  setTimeout(() => {
    const idx = toasts.value.findIndex((t) => t.id === toast.id)
    if (idx > -1) toasts.value.splice(idx, 1)
  }, toast.duration)
}

export const ToastPlugin = {
  install(app: App) {
    // Mount toast container
    const container = document.createElement('div')
    container.id = 'toast-container'
    document.body.appendChild(container)

    const toastApp = createApp(ToastContainer)
    toastApp.mount(container)

    // Global property
    app.config.globalProperties.$toast = showToast

    // Provide for composable use
    app.provide('toast', showToast)
  },
}

// Composable
export function useToast() {
  return showToast
}

// TS augmentation
declare module 'vue' {
  interface ComponentCustomProperties {
    $toast: typeof showToast
  }
}
```

Register:

```typescript
// main.ts
import { createApp } from 'vue'
import { ToastPlugin } from '@/plugins/toast'
import App from './App.vue'

const app = createApp(App)
app.use(ToastPlugin)
app.mount('#app')
```

Usage:

```vue
<script setup lang="ts">
import { useToast } from '@/plugins/toast'

const toast = useToast()

async function save() {
  try {
    await api.save()
    toast('Saved successfully!', { type: 'success' })
  } catch (err) {
    toast(`Error: ${(err as Error).message}`, { type: 'error', duration: 5000 })
  }
}
</script>

<template>
  <button @click="save">Save</button>
  <button @click="$toast('Hello', { type: 'info' })">Info Toast</button>
</template>
```

</details>

---

## Challenge 10: Provide/Inject DI pattern [Middle]

**Talab:** `InjectionKey<T>` bilan typed DI: `provideUser()` provider va `useUser()` consumer composable. Error handling — `useUser` provider tashqarisida throw.

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useUser.ts
import {
  ref,
  computed,
  provide,
  inject,
  type Ref,
  type ComputedRef,
  type InjectionKey,
} from 'vue'

interface User {
  id: number
  name: string
  email: string
  roles: string[]
}

interface UserContext {
  user: Ref<User | null>
  isLoggedIn: ComputedRef<boolean>
  hasRole: (role: string) => boolean
  login: (email: string, password: string) => Promise<User>
  logout: () => Promise<void>
}

const USER_KEY: InjectionKey<UserContext> = Symbol('user-context')

export function provideUser(initialUser: User | null = null): UserContext {
  const user = ref<User | null>(initialUser)

  const isLoggedIn = computed(() => user.value !== null)

  function hasRole(role: string): boolean {
    return user.value?.roles.includes(role) ?? false
  }

  async function login(email: string, password: string): Promise<User> {
    const res = await fetch('/api/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password }),
    })

    if (!res.ok) throw new Error(`Login failed: ${res.status}`)

    const userData: User = await res.json()
    user.value = userData
    return userData
  }

  async function logout(): Promise<void> {
    await fetch('/api/logout', { method: 'POST' })
    user.value = null
  }

  const context: UserContext = { user, isLoggedIn, hasRole, login, logout }
  provide(USER_KEY, context)
  return context
}

export function useUser(): UserContext {
  const context = inject(USER_KEY)
  if (!context) {
    throw new Error(
      '[useUser] Provider not found. Call provideUser() in parent component.'
    )
  }
  return context
}
```

Usage:

```vue
<!-- App.vue (root provider) -->
<script setup lang="ts">
import { provideUser } from '@/composables/useUser'

provideUser()
</script>

<template>
  <RouterView />
</template>
```

```vue
<!-- LoginForm.vue -->
<script setup lang="ts">
import { ref } from 'vue'
import { useUser } from '@/composables/useUser'

const email = ref('')
const password = ref('')
const error = ref<string | null>(null)

const { login, isLoggedIn } = useUser()

async function submit() {
  try {
    await login(email.value, password.value)
  } catch (e) {
    error.value = (e as Error).message
  }
}
</script>

<template>
  <form v-if="!isLoggedIn" @submit.prevent="submit">
    <input v-model="email" type="email" />
    <input v-model="password" type="password" />
    <button>Login</button>
    <p v-if="error" class="error">{{ error }}</p>
  </form>
</template>
```

```vue
<!-- AdminPanel.vue -->
<script setup lang="ts">
import { useUser } from '@/composables/useUser'

const { user, hasRole, logout } = useUser()
</script>

<template>
  <div v-if="hasRole('admin')">
    <h2>Welcome, {{ user?.name }}</h2>
    <button @click="logout">Logout</button>
  </div>
  <p v-else>Access denied</p>
</template>
```

</details>

---

## Challenge 11: Modal komponent (Teleport + defineModel + Transition) [Middle]

**Talab:** Reusable Modal komponent: `v-model:open` bilan boshqarish, `Teleport to="body"`, Transition (fade), Escape key bilan close, backdrop click close.

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- components/AppModal.vue -->
<script setup lang="ts">
import { onMounted, onBeforeUnmount, watch } from 'vue'

const open = defineModel<boolean>('open', { default: false })

const props = withDefaults(defineProps<{
  closable?: boolean
  closeOnBackdrop?: boolean
  closeOnEscape?: boolean
}>(), {
  closable: true,
  closeOnBackdrop: true,
  closeOnEscape: true,
})

const emit = defineEmits<{
  open: []
  close: []
}>()

function close() {
  if (props.closable) {
    open.value = false
  }
}

function onBackdropClick(e: MouseEvent) {
  if (props.closeOnBackdrop && e.target === e.currentTarget) {
    close()
  }
}

function onKeydown(e: KeyboardEvent) {
  if (props.closeOnEscape && e.key === 'Escape' && open.value) {
    close()
  }
}

onMounted(() => {
  window.addEventListener('keydown', onKeydown)
})

onBeforeUnmount(() => {
  window.removeEventListener('keydown', onKeydown)
})

watch(open, (isOpen) => {
  if (isOpen) {
    emit('open')
    document.body.style.overflow = 'hidden'
  } else {
    emit('close')
    document.body.style.overflow = ''
  }
})
</script>

<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="open" class="modal-backdrop" @click="onBackdropClick">
        <div class="modal" role="dialog" aria-modal="true">
          <header v-if="$slots.header || closable" class="modal-header">
            <slot name="header" />
            <button v-if="closable" class="close-btn" @click="close" aria-label="Close">
              ×
            </button>
          </header>

          <div class="modal-body">
            <slot />
          </div>

          <footer v-if="$slots.footer" class="modal-footer">
            <slot name="footer" :close="close" />
          </footer>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<style scoped>
.modal-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal {
  background: white;
  border-radius: 8px;
  box-shadow: 0 25px 50px rgba(0, 0, 0, 0.25);
  max-width: 90vw;
  min-width: 24rem;
  max-height: 90vh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1rem 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}

.modal-body {
  padding: 1.5rem;
  overflow-y: auto;
  flex: 1;
}

.modal-footer {
  padding: 1rem 1.5rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  justify-content: flex-end;
  gap: 0.5rem;
}

.close-btn {
  background: transparent;
  border: 0;
  font-size: 1.5rem;
  cursor: pointer;
  color: #64748b;
}
.close-btn:hover { color: #1e293b; }

/* Transitions */
.modal-enter-active, .modal-leave-active {
  transition: opacity 200ms;
}
.modal-enter-active .modal, .modal-leave-active .modal {
  transition: transform 200ms;
}
.modal-enter-from, .modal-leave-to {
  opacity: 0;
}
.modal-enter-from .modal, .modal-leave-to .modal {
  transform: scale(0.95);
}
</style>
```

Usage:

```vue
<script setup lang="ts">
import { ref } from 'vue'
import AppModal from '@/components/AppModal.vue'

const showModal = ref(false)

function onConfirm() {
  alert('Confirmed!')
  showModal.value = false
}
</script>

<template>
  <button @click="showModal = true">Open Modal</button>

  <AppModal v-model:open="showModal">
    <template #header>
      <h2>Confirm Action</h2>
    </template>

    <p>Are you sure you want to proceed?</p>
    <p>This action cannot be undone.</p>

    <template #footer="{ close }">
      <button @click="close">Cancel</button>
      <button @click="onConfirm">Confirm</button>
    </template>
  </AppModal>
</template>
```

</details>

---

## Challenge 12: Async component + Suspense + Error Boundary [Middle+]

**Talab:** Async component (lazy load), Suspense fallback, ErrorBoundary catch. Retry mexanizm.

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- components/ErrorBoundary.vue -->
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'

const error = ref<Error | null>(null)
const retryKey = ref(0)

onErrorCaptured((err) => {
  error.value = err as Error
  return false
})

function retry() {
  error.value = null
  retryKey.value++ // force re-mount
}

defineExpose({ retry, error })

defineSlots<{
  default(props: { retryKey: number }): unknown
  error?(props: { error: Error; retry: () => void }): unknown
}>()
</script>

<template>
  <slot v-if="!error" :retry-key="retryKey" />

  <slot v-else name="error" :error="error" :retry="retry">
    <!-- Default error UI -->
    <div class="error-boundary">
      <h3>Something went wrong</h3>
      <p>{{ error.message }}</p>
      <button @click="retry">Try again</button>
    </div>
  </slot>
</template>

<style scoped>
.error-boundary {
  padding: 1rem;
  background: #fee2e2;
  border-left: 4px solid #ef4444;
  color: #991b1b;
}
</style>
```

```typescript
// AsyncContent.vue (async setup)
// <script setup lang="ts">
//   const res = await fetch('/api/data')
//   if (!res.ok) throw new Error('Failed to load')
//   const data = await res.json()
// </script>
```

Usage:

```vue
<script setup lang="ts">
import { defineAsyncComponent } from 'vue'
import ErrorBoundary from '@/components/ErrorBoundary.vue'

const AsyncContent = defineAsyncComponent({
  loader: () => import('./AsyncContent.vue'),
  delay: 200,
  timeout: 5000,
})
</script>

<template>
  <ErrorBoundary>
    <template #default="{ retryKey }">
      <Suspense :key="retryKey">
        <AsyncContent />

        <template #fallback>
          <div class="loading">Loading content...</div>
        </template>
      </Suspense>
    </template>

    <template #error="{ error, retry }">
      <div class="error">
        <h3>Failed to load</h3>
        <p>{{ error.message }}</p>
        <button @click="retry">Retry</button>
      </div>
    </template>
  </ErrorBoundary>
</template>
```

Error flow: AsyncContent fetch fails → throw → ErrorBoundary catches → fallback UI. Retry — `retryKey` changes → Suspense re-mounts → fresh fetch.

</details>

---

## Challenge 13: Render function dynamic heading [Senior]

**Talab:** Komponent: `level` prop'iga qarab `<h1>`, `<h2>`, ..., `<h6>` render qilsin. Render function ishlatib (template emas).

<details>
<summary><strong>Javob</strong></summary>

```typescript
// components/DynamicHeading.ts
import { defineComponent, h, computed, type PropType } from 'vue'

type HeadingLevel = 1 | 2 | 3 | 4 | 5 | 6

export default defineComponent({
  name: 'DynamicHeading',
  props: {
    level: {
      type: Number as PropType<HeadingLevel>,
      required: true,
      validator: (v: number) => v >= 1 && v <= 6,
    },
    underline: {
      type: Boolean,
      default: false,
    },
  },
  setup(props, { slots }) {
    const tag = computed(() => `h${props.level}`)

    return () => h(
      tag.value,
      {
        class: ['dynamic-heading', `heading-${props.level}`, { 'has-underline': props.underline }],
      },
      slots.default?.()
    )
  }
})
```

Or pure SFC alternative (template + `<component :is>`):

```vue
<!-- DynamicHeading.vue (template version) -->
<script setup lang="ts">
type HeadingLevel = 1 | 2 | 3 | 4 | 5 | 6

defineProps<{
  level: HeadingLevel
  underline?: boolean
}>()
</script>

<template>
  <component
    :is="`h${level}`"
    class="dynamic-heading"
    :class="[`heading-${level}`, { 'has-underline': underline }]"
  >
    <slot />
  </component>
</template>
```

Usage:

```vue
<script setup lang="ts">
const headings = [
  { level: 1, text: 'Page Title' },
  { level: 2, text: 'Section' },
  { level: 3, text: 'Subsection' },
]
</script>

<template>
  <DynamicHeading
    v-for="h in headings"
    :key="h.text"
    :level="h.level"
    :underline="h.level === 1"
  >
    {{ h.text }}
  </DynamicHeading>
</template>
```

</details>

---

## Challenge 14: Mini reactivity system [Senior]

**Talab:** Vue Reactivity'ning soddalashtirilgan implementation: `reactive`, `effect`, `ref`, `computed`, `watch`. Manual `track`/`trigger` bilan.

<details>
<summary><strong>Javob</strong></summary>

```typescript
// mini-reactivity.ts

// === Helpers ===
// Vue source: hasChanged = !Object.is(value, oldValue) — NaN !== NaN bo'lsa ham
// qayta trigger bo'lmaydi, +0/-0 farqi to'g'ri hisobga olinadi
function hasChanged(value: unknown, oldValue: unknown): boolean {
  return !Object.is(value, oldValue)
}

// === State ===
const targetMap = new WeakMap<object, Map<string | symbol, Set<ReactiveEffect>>>()
let activeEffect: ReactiveEffect | undefined

// === ReactiveEffect ===
class ReactiveEffect {
  active = true
  deps: Set<ReactiveEffect>[] = []
  parent: ReactiveEffect | undefined

  constructor(
    public fn: () => any,
    public scheduler?: () => void,
  ) {}

  run() {
    if (!this.active) return this.fn()

    let parent: ReactiveEffect | undefined = activeEffect
    while (parent) {
      if (parent === this) return
      parent = parent.parent
    }

    try {
      this.parent = activeEffect
      activeEffect = this
      cleanupEffect(this)
      return this.fn()
    } finally {
      activeEffect = this.parent
      this.parent = undefined
    }
  }

  stop() {
    if (this.active) {
      cleanupEffect(this)
      this.active = false
    }
  }
}

function cleanupEffect(effect: ReactiveEffect) {
  for (const dep of effect.deps) {
    dep.delete(effect)
  }
  effect.deps.length = 0
}

// === effect() ===
export function effect(fn: () => any) {
  const _effect = new ReactiveEffect(fn)
  _effect.run()
  return () => _effect.stop()
}

// === track() ===
function track(target: object, key: string | symbol) {
  if (!activeEffect) return

  let depsMap = targetMap.get(target)
  if (!depsMap) targetMap.set(target, (depsMap = new Map()))

  let dep = depsMap.get(key)
  if (!dep) depsMap.set(key, (dep = new Set()))

  if (!dep.has(activeEffect)) {
    dep.add(activeEffect)
    activeEffect.deps.push(dep)
  }
}

// === trigger() ===
function trigger(target: object, key: string | symbol) {
  const depsMap = targetMap.get(target)
  if (!depsMap) return

  const dep = depsMap.get(key)
  if (!dep) return

  const effects = new Set(dep)
  for (const effect of effects) {
    if (effect.scheduler) {
      effect.scheduler()
    } else {
      effect.run()
    }
  }
}

// === reactive() ===
const reactiveMap = new WeakMap()

export function reactive<T extends object>(target: T): T {
  if (typeof target !== 'object' || target === null) return target

  const existing = reactiveMap.get(target)
  if (existing) return existing

  const proxy = new Proxy(target, {
    get(target, key, receiver) {
      const result = Reflect.get(target, key, receiver)
      track(target, key)
      return typeof result === 'object' && result !== null ? reactive(result as object) : result
    },
    set(target, key, value, receiver) {
      const oldValue = Reflect.get(target, key)
      const result = Reflect.set(target, key, value, receiver)
      if (hasChanged(value, oldValue)) trigger(target, key)
      return result
    },
    deleteProperty(target, key) {
      const hadKey = Object.hasOwn(target, key)
      const result = Reflect.deleteProperty(target, key)
      if (hadKey && result) trigger(target, key)
      return result
    },
  })

  reactiveMap.set(target, proxy)
  return proxy as T
}

// === ref() ===
class RefImpl<T> {
  public readonly __v_isRef = true
  private _value: T

  constructor(value: T) {
    this._value = typeof value === 'object' && value !== null
      ? (reactive(value as object) as T)
      : value
  }

  get value() {
    track(this, 'value')
    return this._value
  }

  set value(newVal: T) {
    if (hasChanged(newVal, this._value)) {
      this._value = typeof newVal === 'object' && newVal !== null
        ? (reactive(newVal as object) as T)
        : newVal
      trigger(this, 'value')
    }
  }
}

export function ref<T>(value: T): RefImpl<T> {
  return new RefImpl(value)
}

// === computed() ===
class ComputedRefImpl<T> {
  public readonly __v_isRef = true
  private _value: T | undefined = undefined
  private _dirty = true
  private effect: ReactiveEffect

  constructor(private getter: () => T) {
    this.effect = new ReactiveEffect(getter, () => {
      if (!this._dirty) {
        this._dirty = true
        trigger(this, 'value')
      }
    })
  }

  get value() {
    track(this, 'value')
    if (this._dirty) {
      this._value = this.effect.run()
      this._dirty = false
    }
    return this._value as T
  }
}

export function computed<T>(getter: () => T): ComputedRefImpl<T> {
  return new ComputedRefImpl(getter)
}

// === watch() ===
export function watch<T>(
  source: { value: T } | (() => T),
  callback: (newVal: T, oldVal: T | undefined) => void,
  options: { immediate?: boolean } = {},
) {
  const getter: () => T = typeof source === 'function'
    ? source
    : () => (source as { value: T }).value

  let oldValue: T | undefined

  const _effect = new ReactiveEffect(getter, () => {
    const newValue = _effect.run()
    callback(newValue, oldValue)
    oldValue = newValue
  })

  if (options.immediate) {
    const initialValue = _effect.run()
    callback(initialValue, undefined)
    oldValue = initialValue
  } else {
    oldValue = _effect.run()
  }

  return () => _effect.stop()
}
```

Demo:

```typescript
import { reactive, ref, effect, computed, watch } from './mini-reactivity'

// Reactive object
const state = reactive({ count: 0, name: 'Aziz' })

effect(() => {
  console.log(`State: ${state.count}, ${state.name}`)
})
// Output: "State: 0, Aziz"

state.count++
// Output: "State: 1, Aziz"

// Ref
const score = ref(100)
effect(() => console.log('Score:', score.value))
// Output: "Score: 100"

score.value = 150
// Output: "Score: 150"

// Computed
const doubled = computed(() => score.value * 2)
effect(() => console.log('Doubled:', doubled.value))
// Output: "Doubled: 300"

score.value = 200
// Output: "Score: 200"
// Output: "Doubled: 400"

// Watch
watch(() => state.count, (newVal, oldVal) => {
  console.log(`Count: ${oldVal} → ${newVal}`)
})

state.count = 5
// Output: "Count: 1 → 5"
```

</details>

---

## Challenge 15: TransitionGroup FLIP animation [Middle]

**Talab:** Sortable/shuffle list animatsiya bilan (FLIP technique: First, Last, Invert, Play).

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- AnimatedList.vue -->
<script setup lang="ts">
import { ref } from 'vue'

interface Item {
  id: number
  name: string
}

const items = ref<Item[]>([
  { id: 1, name: 'Apple' },
  { id: 2, name: 'Banana' },
  { id: 3, name: 'Cherry' },
  { id: 4, name: 'Date' },
  { id: 5, name: 'Elderberry' },
])

function shuffle() {
  items.value = [...items.value].sort(() => Math.random() - 0.5)
}

function addItem() {
  const id = Math.max(...items.value.map((i) => i.id)) + 1
  items.value.unshift({ id, name: `New ${id}` })
}

function removeItem(id: number) {
  items.value = items.value.filter((i) => i.id !== id)
}
</script>

<template>
  <div class="list-demo">
    <div class="controls">
      <button @click="shuffle">Shuffle</button>
      <button @click="addItem">Add</button>
    </div>

    <TransitionGroup name="list" tag="ul" class="list">
      <li v-for="item in items" :key="item.id" class="list-item">
        <span>{{ item.name }}</span>
        <button @click="removeItem(item.id)">×</button>
      </li>
    </TransitionGroup>
  </div>
</template>

<style scoped>
.list-demo { padding: 1rem; }
.list {
  list-style: none;
  padding: 0;
  margin: 1rem 0 0;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}
.list-item {
  background: #f1f5f9;
  padding: 0.75rem 1rem;
  border-radius: 6px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

/* FLIP transitions */
.list-enter-active,
.list-leave-active,
.list-move {
  transition: all 400ms ease;
}

/* Enter from / Leave to */
.list-enter-from {
  opacity: 0;
  transform: translateX(-30px);
}

.list-leave-to {
  opacity: 0;
  transform: translateX(30px);
}

/* Leaving element removed from layout flow */
.list-leave-active {
  position: absolute;
  width: calc(100% - 2rem);
}
</style>
```

**Asosiy nuances:**

- `TransitionGroup` `tag` — wrapper element (`<ul>`)
- `:key` — har item unique (FLIP tracking)
- `.list-move` — Vue auto-applies on position change (FLIP magic)
- `.list-leave-active { position: absolute }` — leaving items removed from layout for smooth shuffle

</details>

---

## Challenge 16: `defineCustomElement` Web Component [Senior]

**Talab:** Vue komponent'ni Web Component (Custom Element) sifatida tarqatish: `<app-counter start="10" step="2">`. Vanilla HTML va React'da ham ishlasin.

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- src/elements/Counter.ce.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const props = withDefaults(defineProps<{
  start?: number
  step?: number
}>(), {
  start: 0,
  step: 1,
})

const emit = defineEmits<{
  change: [value: number]
}>()

const count = ref(props.start)

function increment() {
  count.value += props.step
  emit('change', count.value)
}

function decrement() {
  count.value -= props.step
  emit('change', count.value)
}
</script>

<template>
  <div class="counter">
    <button @click="decrement">−</button>
    <span>{{ count }}</span>
    <button @click="increment">+</button>
  </div>
</template>

<style>
:host {
  display: inline-block;
  --counter-bg: #6366f1;
  --counter-text: white;
}

.counter {
  display: inline-flex;
  gap: 0.5rem;
  align-items: center;
  padding: 0.5rem;
  border: 1px solid #cbd5e1;
  border-radius: 8px;
  font-family: system-ui, sans-serif;
}

button {
  padding: 0.5rem 1rem;
  background: var(--counter-bg);
  color: var(--counter-text);
  border: 0;
  border-radius: 4px;
  cursor: pointer;
}

span {
  font-weight: bold;
  min-width: 2rem;
  text-align: center;
}
</style>
```

```typescript
// src/elements.ts
import { defineCustomElement } from 'vue'
import Counter from './elements/Counter.ce.vue'

const CounterElement = defineCustomElement(Counter)

if (!customElements.get('app-counter')) {
  customElements.define('app-counter', CounterElement)
}
```

`vite.config.ts`:

```typescript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [
    vue({
      customElement: /\.ce\.vue$/,
    })
  ],
  build: {
    lib: {
      entry: 'src/elements.ts',
      name: 'AppElements',
      formats: ['es', 'iife'],
      fileName: (format) => `app-elements.${format}.js`,
    },
  }
})
```

Vanilla HTML usage:

```html
<!DOCTYPE html>
<html>
<head>
  <script type="module" src="/dist/app-elements.es.js"></script>
</head>
<body>
  <app-counter start="10" step="2"></app-counter>

  <p id="log">Last change: —</p>

  <script>
    document.querySelector('app-counter').addEventListener('change', (e) => {
      document.getElementById('log').textContent = `Last change: ${e.detail[0]}`
    })
  </script>
</body>
</html>
```

React usage:

```tsx
import { useEffect, useRef, useState } from 'react'

declare namespace JSX {
  interface IntrinsicElements {
    'app-counter': React.DetailedHTMLProps<
      React.HTMLAttributes<HTMLElement> & {
        start?: number
        step?: number
      },
      HTMLElement
    >
  }
}

function App() {
  const ref = useRef<HTMLElement>(null)
  const [lastValue, setLastValue] = useState(0)

  useEffect(() => {
    const el = ref.current
    if (!el) return

    const handler = (e: Event) => {
      const ce = e as CustomEvent<[number]>
      setLastValue(ce.detail[0])
    }

    el.addEventListener('change', handler)
    return () => el.removeEventListener('change', handler)
  }, [])

  return (
    <div>
      <app-counter ref={ref} start={10} step={2} />
      <p>Last value: {lastValue}</p>
    </div>
  )
}
```

</details>

---

## Challenge 17: `defineModel` multi v-model komponent [Middle+]

**Talab:** Form komponent: `firstName`, `lastName`, `email`, `age` v-model'lar. `defineModel('name')` har biri uchun.

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- components/UserForm.vue -->
<script setup lang="ts">
import { computed } from 'vue'

const firstName = defineModel<string>('firstName', { default: '' })
const lastName = defineModel<string>('lastName', { default: '' })
const email = defineModel<string>('email', { default: '' })
const age = defineModel<number>('age', { default: 0 })

const emit = defineEmits<{
  submit: [data: { firstName: string; lastName: string; email: string; age: number }]
}>()

const isValid = computed(() => {
  return firstName.value.length > 0
    && lastName.value.length > 0
    && email.value.includes('@')
    && age.value >= 0
})

function submit() {
  if (!isValid.value) return
  emit('submit', {
    firstName: firstName.value,
    lastName: lastName.value,
    email: email.value,
    age: age.value,
  })
}
</script>

<template>
  <form class="user-form" @submit.prevent="submit">
    <div class="field">
      <label>First Name</label>
      <input v-model="firstName" type="text" placeholder="Aziz" />
    </div>

    <div class="field">
      <label>Last Name</label>
      <input v-model="lastName" type="text" placeholder="Karimov" />
    </div>

    <div class="field">
      <label>Email</label>
      <input v-model="email" type="email" placeholder="aziz@example.com" />
    </div>

    <div class="field">
      <label>Age</label>
      <input v-model.number="age" type="number" placeholder="25" />
    </div>

    <button type="submit" :disabled="!isValid">Submit</button>
  </form>
</template>

<style scoped>
.user-form { display: flex; flex-direction: column; gap: 1rem; max-width: 400px; }
.field { display: flex; flex-direction: column; gap: 0.25rem; }
label { font-weight: 500; }
input { padding: 0.5rem; border: 1px solid #cbd5e1; border-radius: 4px; }
button { padding: 0.75rem; background: #3b82f6; color: white; border: 0; border-radius: 4px; cursor: pointer; }
button:disabled { opacity: 0.5; cursor: not-allowed; }
</style>
```

Usage:

```vue
<script setup lang="ts">
import { reactive } from 'vue'
import UserForm from '@/components/UserForm.vue'

const user = reactive({
  firstName: '',
  lastName: '',
  email: '',
  age: 0,
})

function onSubmit(data: typeof user) {
  console.log('Submitted:', data)
}
</script>

<template>
  <UserForm
    v-model:firstName="user.firstName"
    v-model:lastName="user.lastName"
    v-model:email="user.email"
    v-model:age="user.age"
    @submit="onSubmit"
  />

  <pre>{{ user }}</pre>
</template>
```

Har v-model independent — bir field change → parent state synced, boshqalar ta'sirsiz.

</details>

---

## Challenge 18: Custom renderer (Canvas) [Senior]

**Talab:** Vue custom renderer: VNode tree'ni Canvas'ga render qilsin. `<rect>`, `<circle>`, `<text>` element'lar.

<details>
<summary><strong>Javob</strong></summary>

```typescript
// canvas-renderer.ts
import { createRenderer } from 'vue'

interface CanvasNode {
  type: 'rect' | 'circle' | 'text' | 'group'
  x: number
  y: number
  width?: number
  height?: number
  radius?: number
  text?: string
  color?: string
  children?: CanvasNode[]
  parent?: CanvasNode | null
}

let needsRedraw = false
let rootNode: CanvasNode | null = null
let ctx: CanvasRenderingContext2D | null = null
let canvas: HTMLCanvasElement | null = null

function requestRedraw() {
  if (needsRedraw) return
  needsRedraw = true
  requestAnimationFrame(() => {
    if (ctx && canvas && rootNode) {
      ctx.clearRect(0, 0, canvas.width, canvas.height)
      drawNode(rootNode)
    }
    needsRedraw = false
  })
}

function drawNode(node: CanvasNode) {
  if (!ctx) return

  if (node.type === 'rect') {
    ctx.fillStyle = node.color || 'black'
    ctx.fillRect(node.x, node.y, node.width || 50, node.height || 50)
  } else if (node.type === 'circle') {
    ctx.fillStyle = node.color || 'black'
    ctx.beginPath()
    ctx.arc(node.x, node.y, node.radius || 25, 0, Math.PI * 2)
    ctx.fill()
  } else if (node.type === 'text') {
    ctx.fillStyle = node.color || 'black'
    ctx.font = '16px sans-serif'
    ctx.fillText(node.text || '', node.x, node.y)
  }

  if (node.children) {
    for (const child of node.children) {
      drawNode(child)
    }
  }
}

const { createApp } = createRenderer<CanvasNode, CanvasNode>({
  createElement(tag) {
    return {
      type: tag as CanvasNode['type'],
      x: 0,
      y: 0,
      children: [],
      parent: null,
    }
  },

  createText() {
    return { type: 'text', x: 0, y: 0, text: '', children: [], parent: null }
  },

  createComment() {
    return { type: 'group', x: 0, y: 0, children: [], parent: null }
  },

  setText(node, text) {
    node.text = text
    requestRedraw()
  },

  setElementText(node, text) {
    node.children = [{ type: 'text', x: 0, y: 0, text, parent: node }]
    requestRedraw()
  },

  insert(child, parent, anchor) {
    child.parent = parent
    if (!parent.children) parent.children = []

    if (anchor) {
      const idx = parent.children.indexOf(anchor)
      parent.children.splice(idx, 0, child)
    } else {
      parent.children.push(child)
    }

    requestRedraw()
  },

  remove(child) {
    const parent = child.parent
    if (parent?.children) {
      const idx = parent.children.indexOf(child)
      if (idx > -1) parent.children.splice(idx, 1)
    }
    child.parent = null
    requestRedraw()
  },

  parentNode(node) {
    return node.parent || null
  },

  nextSibling(node) {
    if (!node.parent?.children) return null
    const idx = node.parent.children.indexOf(node)
    return node.parent.children[idx + 1] || null
  },

  patchProp(el, key, _prev, next) {
    // width/height/radius/color/text — createElement'da initialize qilinmagan,
    // shuning uchun `key in el` tekshiruvi bo'lmaydi: prop'ni to'g'ridan yozamiz
    ;(el as Record<string, unknown>)[key] = next
    requestRedraw()
  },
})

export function mountCanvasApp(component: Parameters<typeof createApp>[0], container: HTMLCanvasElement) {
  canvas = container
  ctx = container.getContext('2d')
  rootNode = { type: 'group', x: 0, y: 0, children: [], parent: null }

  const app = createApp(component)
  app.mount(rootNode)

  return app
}
```

Usage:

```vue
<!-- CanvasScene.vue -->
<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue'
import { mountCanvasApp } from './canvas-renderer'

const ballX = ref(50)
const ballY = ref(50)
let animationId: number

function animate() {
  ballX.value = 200 + Math.sin(Date.now() / 500) * 100
  ballY.value = 100 + Math.cos(Date.now() / 500) * 50

  animationId = requestAnimationFrame(animate)
}

onMounted(() => {
  animate()
})

onBeforeUnmount(() => {
  cancelAnimationFrame(animationId)
})
</script>

<template>
  <group>
    <rect :x="10" :y="10" :width="100" :height="50" color="#3b82f6" />
    <circle :x="ballX" :y="ballY" :radius="20" color="#ef4444" />
    <text :x="10" :y="200" :text="`Ball position: ${ballX.toFixed(0)}, ${ballY.toFixed(0)}`" color="#1e293b" />
  </group>
</template>
```

Main entry:

```typescript
// main.ts
import { mountCanvasApp } from './canvas-renderer'
import CanvasScene from './CanvasScene.vue'

const canvas = document.querySelector<HTMLCanvasElement>('#canvas')
if (!canvas) throw new Error('Canvas element not found')
canvas.width = 800
canvas.height = 400

mountCanvasApp(CanvasScene, canvas)
```

```html
<!DOCTYPE html>
<html>
<body>
  <canvas id="canvas"></canvas>
  <script type="module" src="/src/main.ts"></script>
</body>
</html>
```

Vue komponent reactive state → Canvas redraw automatic. `ballX.value` change → patchProp → requestRedraw → animation frame.

`vite.config.ts` — `isCustomElement` configure (compiler shouldn't try resolve `rect`/`circle`/`text`):

```typescript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [
    vue({
      template: {
        compilerOptions: {
          isCustomElement: (tag) => ['rect', 'circle', 'text', 'group'].includes(tag)
        }
      }
    })
  ]
})
```

</details>

---

## Challenge 19: `useToggle` — boolean state management [Junior+]

**Vazifa:** Boolean state'ni toggle qiladigan composable. `value`, `toggle`, `setTrue`, `setFalse` qaytarsin.

**Talablar:**
- TypeScript strict
- Initial value parametr sifatida qabul qilsin
- `toggle()` — qiymatni teskari qilsin
- `setTrue()` / `setFalse()` — aniq set qilsin

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useToggle.ts
import { ref, type Ref } from 'vue'

interface UseToggleReturn {
  value: Ref<boolean>
  toggle: () => void
  setTrue: () => void
  setFalse: () => void
}

export function useToggle(initialValue: boolean = false): UseToggleReturn {
  const value = ref(initialValue)

  function toggle() {
    value.value = !value.value
  }

  function setTrue() {
    value.value = true
  }

  function setFalse() {
    value.value = false
  }

  return { value, toggle, setTrue, setFalse }
}
```

Usage:

```vue
<script setup lang="ts">
import { useToggle } from '@/composables/useToggle'

const { value: isDarkMode, toggle: toggleTheme } = useToggle(false)
const { value: isSidebarOpen, toggle: toggleSidebar, setFalse: closeSidebar } = useToggle(true)
</script>

<template>
  <div :class="{ dark: isDarkMode }">
    <button @click="toggleTheme">
      {{ isDarkMode ? 'Light Mode' : 'Dark Mode' }}
    </button>

    <aside v-if="isSidebarOpen">
      <nav>Sidebar content</nav>
      <button @click="closeSidebar">Close</button>
    </aside>
    <button @click="toggleSidebar">Toggle Sidebar</button>
  </div>
</template>
```

**Tushuntirish:**
- Sodda composable, lekin production'da ko'p takrorlanadigan pattern'ni abstrakt qiladi
- Destructuring paytida `value` nomi alias qilinadi (`value: isDarkMode`) — bir komponentda bir nechta toggle ishlatish uchun
- `setTrue`/`setFalse` event handler'larda inline arrow function'siz ishlatiladi

</details>

---

## Challenge 20: `useLocalStorage<T>` — SSR-safe typed storage [Middle+]

**Vazifa:** `localStorage` bilan synced reactive ref yarating. SSR-safe, JSON serialize/deserialize, type-safe.

**Talablar:**
- Generic type `<T>` — string, number, object, array
- SSR-safe (server'da `localStorage` yo'q — fallback default value)
- JSON parse/stringify error handling
- `storage` event bilan boshqa tab'lar bilan sync
- Cleanup `onBeforeUnmount`'da

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useLocalStorage.ts
import { ref, watch, onBeforeUnmount, type Ref } from 'vue'

export function useLocalStorage<T>(
  key: string,
  defaultValue: T,
): Ref<T> {
  const isClient = typeof window !== 'undefined'

  function read(): T {
    if (!isClient) return defaultValue

    try {
      const raw = localStorage.getItem(key)
      if (raw === null) return defaultValue
      return JSON.parse(raw) as T
    } catch {
      return defaultValue
    }
  }

  function write(value: T): void {
    if (!isClient) return

    try {
      if (value === null || value === undefined) {
        localStorage.removeItem(key)
      } else {
        localStorage.setItem(key, JSON.stringify(value))
      }
    } catch (error) {
      console.warn(`[useLocalStorage] Failed to write key "${key}":`, error)
    }
  }

  const data = ref<T>(read()) as Ref<T>

  // Persist on change
  watch(data, (newValue) => {
    write(newValue)
  }, { deep: true })

  // Cross-tab sync
  function onStorageChange(event: StorageEvent) {
    if (event.key !== key) return

    if (event.newValue === null) {
      data.value = defaultValue
    } else {
      try {
        data.value = JSON.parse(event.newValue) as T
      } catch {
        data.value = defaultValue
      }
    }
  }

  if (isClient) {
    window.addEventListener('storage', onStorageChange)
  }

  onBeforeUnmount(() => {
    if (isClient) {
      window.removeEventListener('storage', onStorageChange)
    }
  })

  return data
}
```

Usage:

```vue
<script setup lang="ts">
import { useLocalStorage } from '@/composables/useLocalStorage'

interface UserPreferences {
  theme: 'light' | 'dark'
  fontSize: number
  locale: string
}

const preferences = useLocalStorage<UserPreferences>('user-preferences', {
  theme: 'light',
  fontSize: 16,
  locale: 'en',
})

const recentSearches = useLocalStorage<string[]>('recent-searches', [])

function addSearch(query: string) {
  const searches = [...recentSearches.value]
  const index = searches.indexOf(query)
  if (index > -1) searches.splice(index, 1)
  searches.unshift(query)
  recentSearches.value = searches.slice(0, 10) // oxirgi 10 ta
}
</script>

<template>
  <div>
    <select v-model="preferences.theme">
      <option value="light">Light</option>
      <option value="dark">Dark</option>
    </select>

    <input
      v-model.number="preferences.fontSize"
      type="range"
      min="12"
      max="24"
    />
    <span>{{ preferences.fontSize }}px</span>
  </div>
</template>
```

**Tushuntirish:**
- `typeof window !== 'undefined'` — SSR guard. Nuxt va Next framework'larida server render paytida `localStorage` mavjud emas
- `watch` `deep: true` — nested object property o'zgarganda ham persist qiladi
- `storage` event faqat **boshqa** tab'dan trigger bo'ladi (joriy tab emas) — cross-tab sync uchun
- `try/catch` — corrupt data, QuotaExceeded va private browsing rejimi holatlarini handle qiladi

</details>

---

## Challenge 21: `useFetch<T>` — abort + loading + error + refetch [Middle+]

**Vazifa:** HTTP request composable: reactive `data`, `error`, `isLoading`. AbortController bilan cancel, `refetch` funksiya.

**Talablar:**
- Generic type `<T>` — response type
- AbortController — component unmount yoki yangi request'da avvalgisini cancel
- `isLoading`, `error`, `data` reactive ref'lar
- `refetch()` — manual re-request
- URL reactive bo'lsa (Ref/computed), auto-refetch

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useFetch.ts
import {
  ref,
  watch,
  unref,
  onBeforeUnmount,
  isRef,
  type Ref,
  type MaybeRef,
} from 'vue'

interface UseFetchOptions {
  immediate?: boolean
  method?: 'GET' | 'POST' | 'PUT' | 'DELETE'
  headers?: Record<string, string>
  body?: unknown
}

interface UseFetchReturn<T> {
  data: Ref<T | null>
  error: Ref<Error | null>
  isLoading: Ref<boolean>
  refetch: () => Promise<void>
  abort: () => void
}

export function useFetch<T>(
  url: MaybeRef<string>,
  options: UseFetchOptions = {},
): UseFetchReturn<T> {
  const { immediate = true, method = 'GET', headers, body } = options

  const data = ref<T | null>(null) as Ref<T | null>
  const error = ref<Error | null>(null)
  const isLoading = ref(false)

  let abortController: AbortController | null = null

  function abort() {
    if (abortController) {
      abortController.abort()
      abortController = null
    }
  }

  async function execute(): Promise<void> {
    const requestUrl = unref(url)
    if (!requestUrl) return

    // Cancel previous request
    abort()
    abortController = new AbortController()

    isLoading.value = true
    error.value = null

    try {
      const response = await fetch(requestUrl, {
        method,
        headers: {
          'Content-Type': 'application/json',
          ...headers,
        },
        body: body ? JSON.stringify(body) : undefined,
        signal: abortController.signal,
      })

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`)
      }

      data.value = (await response.json()) as T
    } catch (err) {
      if (err instanceof DOMException && err.name === 'AbortError') {
        return // Request cancelled — error state'ga yozmaslik
      }
      error.value = err instanceof Error ? err : new Error(String(err))
    } finally {
      isLoading.value = false
    }
  }

  // URL ref bo'lsa, auto-refetch
  if (isRef(url)) {
    watch(url, () => execute())
  }

  if (immediate) {
    execute()
  }

  onBeforeUnmount(abort)

  return {
    data,
    error,
    isLoading,
    refetch: execute,
    abort,
  }
}
```

Usage:

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'
import { useFetch } from '@/composables/useFetch'

interface Product {
  id: number
  title: string
  price: number
  category: string
}

const categoryId = ref(1)
const apiUrl = computed(() => `/api/products?category=${categoryId.value}`)

const { data: products, isLoading, error, refetch } = useFetch<Product[]>(apiUrl)
</script>

<template>
  <div>
    <select v-model="categoryId">
      <option :value="1">Electronics</option>
      <option :value="2">Clothing</option>
    </select>

    <button @click="refetch" :disabled="isLoading">
      {{ isLoading ? 'Loading...' : 'Refresh' }}
    </button>

    <p v-if="error" class="error">{{ error.message }}</p>

    <ul v-if="products">
      <li v-for="product in products" :key="product.id">
        {{ product.title }} — ${{ product.price }}
      </li>
    </ul>
  </div>
</template>
```

**Tushuntirish:**
- AbortController har yangi request'da avvalgisini cancel qiladi — race condition oldini oladi (eski response yangi request'dan keyin kelishi mumkin)
- `onBeforeUnmount(abort)` — component unmount bo'lganda in-flight request'ni cancel qiladi (memory leak prevention)
- `isRef(url)` tekshiruvi — faqat reactive URL bo'lganda watch qo'shiladi, static string'da ortiqcha watcher yo'q
- `AbortError` alohida handle qilinadi — cancel qilingan request xato emas

</details>

---

## Challenge 22: `useOnClickOutside` — multiple elements [Middle]

**Vazifa:** Element tashqarisida click bo'lganda handler chaqiradigan composable. Bir nechta element'ni ignore qilish mumkin.

**Talablar:**
- Target element + optional ignore elements array
- Pointer event va touch support
- Cleanup `onBeforeUnmount`'da
- `useTemplateRef` bilan ishlashi kerak

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useOnClickOutside.ts
import { onBeforeUnmount, onMounted, unref, type Ref } from 'vue'

type MaybeElementRef = Ref<HTMLElement | null> | HTMLElement | null

export function useOnClickOutside(
  target: MaybeElementRef,
  handler: (event: PointerEvent) => void,
  options: {
    ignore?: MaybeElementRef[]
  } = {},
) {
  function listener(event: PointerEvent) {
    const el = unref(target)
    if (!el) return

    // Target ichida click — ignore
    if (el.contains(event.target as Node)) return

    // Ignore list'dagi element'lar ichida click — ignore
    if (options.ignore?.length) {
      for (const ignoreRef of options.ignore) {
        const ignoreEl = unref(ignoreRef)
        if (ignoreEl && ignoreEl.contains(event.target as Node)) return
      }
    }

    handler(event)
  }

  onMounted(() => {
    document.addEventListener('pointerdown', listener)
  })

  onBeforeUnmount(() => {
    document.removeEventListener('pointerdown', listener)
  })
}
```

Usage:

```vue
<script setup lang="ts">
import { ref, useTemplateRef } from 'vue'
import { useOnClickOutside } from '@/composables/useOnClickOutside'

const isDropdownOpen = ref(false)
const dropdownRef = useTemplateRef<HTMLDivElement>('dropdown')
const triggerRef = useTemplateRef<HTMLButtonElement>('trigger')

useOnClickOutside(dropdownRef, () => {
  isDropdownOpen.value = false
}, {
  ignore: [triggerRef], // trigger button'ga bosganda close qilmaslik
})
</script>

<template>
  <div class="dropdown-wrapper">
    <button ref="trigger" @click="isDropdownOpen = !isDropdownOpen">
      Menu
    </button>

    <div v-if="isDropdownOpen" ref="dropdown" class="dropdown-panel">
      <a href="/profile">Profile</a>
      <a href="/settings">Settings</a>
      <a href="/logout">Logout</a>
    </div>
  </div>
</template>
```

**Tushuntirish:**
- `pointerdown` — `click`'dan farqli, pointer event mouse va touch'ni birlashtiradi
- `ignore` array — trigger button tashqarida tursa ham, unga bosganda dropdown close bo'lmasligi kerak
- `contains()` — DOM API, child element'larni ham tekshiradi

</details>

---

## Challenge 23: `useWindowSize` — SSR-safe + debounce [Middle]

**Vazifa:** Window size'ni reactive qaytaradigan composable. SSR-safe, debounce option.

**Talablar:**
- `width` va `height` reactive ref'lar
- SSR-safe — server'da default value
- Optional debounce (resize event juda tez fire bo'ladi)
- Cleanup

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useWindowSize.ts
import { ref, onMounted, onBeforeUnmount } from 'vue'

interface UseWindowSizeOptions {
  debounceMs?: number
  initialWidth?: number
  initialHeight?: number
}

export function useWindowSize(options: UseWindowSizeOptions = {}) {
  const {
    debounceMs = 100,
    initialWidth = 0,
    initialHeight = 0,
  } = options

  const isClient = typeof window !== 'undefined'
  const width = ref(isClient ? window.innerWidth : initialWidth)
  const height = ref(isClient ? window.innerHeight : initialHeight)

  let timeoutId: ReturnType<typeof setTimeout> | null = null

  function update() {
    width.value = window.innerWidth
    height.value = window.innerHeight
  }

  function onResize() {
    if (debounceMs <= 0) {
      update()
      return
    }

    if (timeoutId) clearTimeout(timeoutId)
    timeoutId = setTimeout(update, debounceMs)
  }

  onMounted(() => {
    update() // initial value
    window.addEventListener('resize', onResize, { passive: true })
  })

  onBeforeUnmount(() => {
    window.removeEventListener('resize', onResize)
    if (timeoutId) clearTimeout(timeoutId)
  })

  return { width, height }
}
```

Usage:

```vue
<script setup lang="ts">
import { computed } from 'vue'
import { useWindowSize } from '@/composables/useWindowSize'

const { width, height } = useWindowSize({ debounceMs: 150 })

const breakpoint = computed(() => {
  if (width.value < 640) return 'mobile'
  if (width.value < 1024) return 'tablet'
  return 'desktop'
})
</script>

<template>
  <div>
    <p>{{ width }} x {{ height }}</p>
    <p>Breakpoint: {{ breakpoint }}</p>
  </div>
</template>
```

**Tushuntirish:**
- `{ passive: true }` — resize event handler'da DOM mutation yo'q, browser optimize qiladi
- Debounce — resize event drag paytida uzluksiz ketma-ket fire bo'ladi, har birida state update qilish ortiqcha render'ga olib keladi; debounce oxirgi event'dan keyin bir marta update qiladi
- SSR guard — Nuxt'da server render paytida `window` mavjud emas, `initialWidth`/`initialHeight` fallback sifatida

</details>

---

## Challenge 24: `useClipboard` — Clipboard API + fallback [Middle]

**Vazifa:** Clipboard'ga text copy qiladigan composable. Modern Clipboard API + legacy fallback.

**Talablar:**
- `copy(text)` — clipboard'ga yozish
- `copiedText` — oxirgi copy qilingan text
- `isSupported` — Clipboard API support
- Auto-reset (copy qilingandan keyin N ms'dan keyin `copiedText` tozalansin)

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useClipboard.ts
import { ref, type Ref } from 'vue'

interface UseClipboardOptions {
  resetAfterMs?: number
}

interface UseClipboardReturn {
  copy: (text: string) => Promise<void>
  copiedText: Ref<string>
  isSupported: boolean
}

export function useClipboard(
  options: UseClipboardOptions = {},
): UseClipboardReturn {
  const { resetAfterMs = 2000 } = options

  const copiedText = ref('')
  const isSupported = typeof navigator !== 'undefined'
    && 'clipboard' in navigator

  let resetTimeoutId: ReturnType<typeof setTimeout> | null = null

  async function copy(text: string): Promise<void> {
    if (resetTimeoutId) clearTimeout(resetTimeoutId)

    try {
      if (isSupported) {
        await navigator.clipboard.writeText(text)
      } else {
        legacyCopy(text)
      }

      copiedText.value = text

      if (resetAfterMs > 0) {
        resetTimeoutId = setTimeout(() => {
          copiedText.value = ''
        }, resetAfterMs)
      }
    } catch (error) {
      console.warn('[useClipboard] Copy failed:', error)
      // Clipboard API permission denied — legacy fallback
      legacyCopy(text)
      copiedText.value = text
    }
  }

  return { copy, copiedText, isSupported }
}

function legacyCopy(text: string): void {
  const textarea = document.createElement('textarea')
  textarea.value = text
  textarea.style.cssText = 'position:fixed;left:-9999px;top:-9999px'
  document.body.appendChild(textarea)
  textarea.select()
  document.execCommand('copy')
  document.body.removeChild(textarea)
}
```

Usage:

```vue
<script setup lang="ts">
import { useClipboard } from '@/composables/useClipboard'

const { copy, copiedText } = useClipboard({ resetAfterMs: 3000 })

const inviteLink = 'https://app.example.com/invite/abc123'
</script>

<template>
  <div class="invite-section">
    <input :value="inviteLink" readonly />
    <button @click="copy(inviteLink)">
      {{ copiedText ? 'Copied' : 'Copy Link' }}
    </button>
  </div>
</template>
```

**Tushuntirish:**
- `navigator.clipboard.writeText` — modern async API, HTTPS yoki localhost talab qiladi
- Legacy fallback — eski browser'lar uchun `document.execCommand('copy')` (deprecated, lekin keng qo'llab-quvvatlanadi)
- Permission denied holati — Clipboard API user gesture talab qiladi, aks holda reject bo'ladi, shu paytda legacy'ga fallback
- `resetAfterMs` — "Copied!" feedback'ni vaqtinchalik ko'rsatish uchun

</details>

---

## Challenge 25: `usePrevious<T>` — oldingi qiymat [Junior+]

**Vazifa:** Reactive value'ning oldingi qiymatini saqlaydigan composable.

**Talablar:**
- Generic type `<T>`
- Har watch trigger'da oldingi qiymat saqlansin
- Initial value `undefined`

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/usePrevious.ts
import { ref, watch, type Ref } from 'vue'

export function usePrevious<T>(
  source: Ref<T> | (() => T),
): Ref<T | undefined> {
  const previous = ref<T | undefined>(undefined) as Ref<T | undefined>

  watch(source, (_newValue, oldValue) => {
    previous.value = oldValue
  })

  return previous
}
```

Usage:

```vue
<script setup lang="ts">
import { ref, watch } from 'vue'
import { usePrevious } from '@/composables/usePrevious'

const count = ref(0)
const previousCount = usePrevious(count)

const selectedTab = ref('home')
const previousTab = usePrevious(selectedTab)

watch(selectedTab, (current) => {
  console.log(`Tab: ${previousTab.value} -> ${current}`)
})
</script>

<template>
  <div>
    <button @click="count++">Count: {{ count }}</button>
    <p>Previous: {{ previousCount ?? 'none' }}</p>

    <nav>
      <button
        v-for="tab in ['home', 'profile', 'settings']"
        :key="tab"
        @click="selectedTab = tab"
        :class="{ active: selectedTab === tab }"
      >
        {{ tab }}
      </button>
    </nav>
    <p v-if="previousTab">Last tab: {{ previousTab }}</p>
  </div>
</template>
```

**Tushuntirish:**
- `watch` callback'ning ikkinchi parametri (`oldValue`) avtomatik oldingi qiymatni beradi
- Vue `watch` deep comparison qilmaydi (default), shuning uchun primitive value'lar uchun optimal
- Object/array track qilish kerak bo'lsa, `watch` `{ deep: true }` qo'shish mumkin

</details>

---

## Challenge 26: `v-lazy-load` — IntersectionObserver image loading [Middle+]

**Vazifa:** Image'larni viewport'ga kirgandagina yuklash uchun custom directive.

**Talablar:**
- `v-lazy-load="'/path/to/image.jpg'"` syntax
- IntersectionObserver — viewport'ga yaqinlashganda load
- Placeholder (low-quality yoki skeleton) to'g'ri load bo'lguncha
- Load bo'lganda observer disconnect
- `rootMargin` bilan prefetch (viewport'dan oldin load boshlash)

<details>
<summary><strong>Javob</strong></summary>

```typescript
// directives/vLazyLoad.ts
import type { Directive } from 'vue'

interface LazyLoadEl extends HTMLImageElement {
  _lazyObserver?: IntersectionObserver
}

export const vLazyLoad: Directive<LazyLoadEl, string> = {
  mounted(el, binding) {
    const src = binding.value
    if (!src) return

    // Placeholder — element'ning mavjud src'si yoki transparent pixel
    if (!el.src) {
      el.src = 'data:image/svg+xml,%3Csvg xmlns="http://www.w3.org/2000/svg" width="1" height="1"%3E%3C/svg%3E'
    }

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (!entry.isIntersecting) return

        // Load real image
        el.src = src
        el.removeAttribute('loading')

        // Error handling
        el.onerror = () => {
          el.alt = el.alt || 'Image failed to load'
          el.classList.add('lazy-error')
        }

        el.onload = () => {
          el.classList.add('lazy-loaded')
        }

        // Bir marta load — disconnect
        observer.disconnect()
      },
      {
        rootMargin: binding.arg === 'eager' ? '500px' : '200px',
        threshold: 0,
      },
    )

    observer.observe(el)
    el._lazyObserver = observer
  },

  updated(el, binding) {
    if (binding.value !== binding.oldValue && el.classList.contains('lazy-loaded')) {
      el.src = binding.value
    }
  },

  beforeUnmount(el) {
    if (el._lazyObserver) {
      el._lazyObserver.disconnect()
      delete el._lazyObserver
    }
  },
}
```

Usage:

```vue
<script setup lang="ts">
interface GalleryImage {
  id: number
  url: string
  alt: string
}

const images: GalleryImage[] = [
  { id: 1, url: '/photos/landscape-01.jpg', alt: 'Mountain landscape' },
  { id: 2, url: '/photos/landscape-02.jpg', alt: 'Ocean sunset' },
  { id: 3, url: '/photos/landscape-03.jpg', alt: 'Forest trail' },
]
</script>

<template>
  <div class="gallery">
    <img
      v-for="image in images"
      :key="image.id"
      v-lazy-load="image.url"
      :alt="image.alt"
      class="gallery-image"
    />
  </div>
</template>

<style scoped>
.gallery-image {
  width: 100%;
  aspect-ratio: 16 / 9;
  object-fit: cover;
  background: #e2e8f0;
  transition: opacity 300ms;
}
.gallery-image:not(.lazy-loaded) { opacity: 0.5; }
.gallery-image.lazy-loaded { opacity: 1; }
.gallery-image.lazy-error { border: 2px solid #ef4444; }
</style>
```

**Tushuntirish:**
- `rootMargin: '200px'` — viewport'dan 200px oldin load boshlaydi (user scroll qilganida image tayyor bo'ladi)
- `binding.arg === 'eager'` — `v-lazy-load:eager` modifier bilan 500px margin qo'yiladi (critical content uchun)
- Observer load bo'lgandan keyin `disconnect` qilinadi — keraksiz observation to'xtaydi
- SVG placeholder — layout shift oldini oladi (image o'lchami CSS'da belgilangan)

</details>

---

## Challenge 27: `v-permission` — role-based visibility [Middle]

**Vazifa:** Element'ni user role'iga qarab ko'rsatish/yashirish uchun custom directive.

**Talablar:**
- `v-permission="'admin'"` — faqat admin ko'rsin
- Array support: `v-permission="['admin', 'editor']"` — ikkalasidan biri
- Provider'dan user roles olish (inject)
- Role yo'q bo'lsa element DOM'dan olib tashlansin

<details>
<summary><strong>Javob</strong></summary>

```typescript
// directives/vPermission.ts
import type { Directive, DirectiveBinding } from 'vue'

interface PermissionComment extends Comment {
  _permissionEl?: HTMLElement
  _permissionBinding?: DirectiveBinding<string | string[]>
}

// User context'dan roles olish uchun global function
// Real loyihada inject/provide yoki store ishlatiladi
let getRoles: () => string[] = () => []

export function setPermissionResolver(resolver: () => string[]) {
  getRoles = resolver
}

export const vPermission: Directive<HTMLElement, string | string[]> = {
  mounted(el, binding) {
    checkPermission(el, binding)
  },

  updated(el, binding) {
    checkPermission(el, binding)
  },
}

function checkPermission(el: HTMLElement, binding: DirectiveBinding<string | string[]>) {
  const requiredRoles = Array.isArray(binding.value)
    ? binding.value
    : [binding.value]

  const userRoles = getRoles()
  const hasPermission = requiredRoles.some((role) => userRoles.includes(role))

  if (!hasPermission) {
    // Comment node bilan almashtirish (v-if pattern)
    const comment: PermissionComment = document.createComment('v-permission')
    el.parentNode?.replaceChild(comment, el)

    // Restore uchun saqlash
    comment._permissionEl = el
    comment._permissionBinding = binding
  }
}
```

Plugin sifatida register:

```typescript
// plugins/permission.ts
import type { App } from 'vue'
import { vPermission, setPermissionResolver } from '@/directives/vPermission'

export function createPermissionPlugin(getRoles: () => string[]) {
  return {
    install(app: App) {
      setPermissionResolver(getRoles)
      app.directive('permission', vPermission)
    },
  }
}
```

```typescript
// main.ts
import { createApp } from 'vue'
import { createPermissionPlugin } from '@/plugins/permission'
import { useAuthStore } from '@/stores/auth'

const app = createApp(App)

const permissionPlugin = createPermissionPlugin(() => {
  const auth = useAuthStore()
  return auth.user?.roles ?? []
})

app.use(permissionPlugin)
```

Usage:

```vue
<template>
  <div class="admin-panel">
    <h1>Dashboard</h1>

    <!-- Faqat admin ko'radi -->
    <button v-permission="'admin'" @click="deleteAllData">
      Delete All Data
    </button>

    <!-- Admin yoki editor ko'radi -->
    <section v-permission="['admin', 'editor']">
      <h2>Content Management</h2>
      <button>Create Post</button>
      <button>Edit Post</button>
    </section>

    <!-- Hamma ko'radi (directive yo'q) -->
    <section>
      <h2>Public Dashboard</h2>
    </section>
  </div>
</template>
```

**Tushuntirish:**
- Element DOM'dan olib tashlanadi (CSS `display: none` emas) — security uchun, DevTools'da ko'rinmaydi
- `comment` node bilan almashtiriladi — Vue `v-if` ham `false` branch'da comment anchor node qoldiradi, bu yerda shu mexanizm qo'lda takrorlanadi
- `some()` — requiredRoles'dan kamida bittasi userRoles'da bo'lsa yetarli (OR logic)
- Plugin pattern — roles source'ni configure qilish mumkin (Pinia store, composable, API)
- **Cheklov:** ushbu sodda implementation role keyin o'zgarsa (masalan, user role oldi) elementni qayta tiklamaydi — element DOM'dan olib tashlangach `updated` hook umuman chaqirilmaydi (komponent endi bu DOM tugunini track qilmaydi). Production'da `v-if="hasRole(...)"` kompozitsion variant ishonchli

</details>

---

## Challenge 28: `v-longpress` — long press event [Middle+]

**Vazifa:** Element'da uzoq bosish (long press) event'ini trigger qiladigan custom directive.

**Talablar:**
- `v-longpress="handler"` — default 500ms keyin fire
- `v-longpress:1000="handler"` — custom delay (ms)
- Touch va mouse support
- Movement threshold — barmoq/mouse harakat qilsa cancel
- Cleanup

<details>
<summary><strong>Javob</strong></summary>

```typescript
// directives/vLongpress.ts
import type { Directive } from 'vue'

interface LongpressEl extends HTMLElement {
  _longpressCleanup?: () => void
}

const MOVE_THRESHOLD = 10 // px — bu qiymatdan oshsa cancel

export const vLongpress: Directive<LongpressEl, (event: PointerEvent) => void> = {
  mounted(el, binding) {
    const delay = binding.arg ? parseInt(binding.arg, 10) : 500
    const handler = binding.value

    if (typeof handler !== 'function') {
      console.warn('[v-longpress] expects a function value')
      return
    }

    let timeoutId: ReturnType<typeof setTimeout> | null = null
    let startX = 0
    let startY = 0
    let fired = false

    function onPointerDown(event: PointerEvent) {
      startX = event.clientX
      startY = event.clientY
      fired = false

      // Capture pointer — pointer leave bo'lsa ham track qilish
      el.setPointerCapture(event.pointerId)

      timeoutId = setTimeout(() => {
        fired = true
        handler(event)
      }, delay)
    }

    function onPointerMove(event: PointerEvent) {
      if (!timeoutId) return

      const dx = event.clientX - startX
      const dy = event.clientY - startY
      const distance = Math.sqrt(dx * dx + dy * dy)

      if (distance > MOVE_THRESHOLD) {
        cancel()
      }
    }

    function onPointerUp(event: PointerEvent) {
      cancel()
      // Long press fire bo'lgan bo'lsa, click event'ni suppress qilish
      if (fired) {
        event.preventDefault()
      }
    }

    function cancel() {
      if (timeoutId) {
        clearTimeout(timeoutId)
        timeoutId = null
      }
    }

    el.addEventListener('pointerdown', onPointerDown)
    el.addEventListener('pointermove', onPointerMove)
    el.addEventListener('pointerup', onPointerUp)
    el.addEventListener('pointercancel', cancel)

    // Context menu suppress (mobile long press)
    function onContextMenu(event: Event) {
      if (fired) event.preventDefault()
    }
    el.addEventListener('contextmenu', onContextMenu)

    el._longpressCleanup = () => {
      cancel()
      el.removeEventListener('pointerdown', onPointerDown)
      el.removeEventListener('pointermove', onPointerMove)
      el.removeEventListener('pointerup', onPointerUp)
      el.removeEventListener('pointercancel', cancel)
      el.removeEventListener('contextmenu', onContextMenu)
    }
  },

  beforeUnmount(el) {
    if (el._longpressCleanup) {
      el._longpressCleanup()
      delete el._longpressCleanup
    }
  },
}
```

Usage:

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface Message {
  id: number
  text: string
  sender: string
}

const messages = ref<Message[]>([
  { id: 1, text: 'Project deadline moved to Friday', sender: 'Aziz' },
  { id: 2, text: 'Code review completed', sender: 'Malika' },
])

const selectedMessage = ref<Message | null>(null)
const showContextMenu = ref(false)

function onLongPress(message: Message) {
  selectedMessage.value = message
  showContextMenu.value = true
}
</script>

<template>
  <ul class="message-list">
    <li
      v-for="message in messages"
      :key="message.id"
      v-longpress="() => onLongPress(message)"
      class="message"
    >
      <strong>{{ message.sender }}:</strong> {{ message.text }}
    </li>
  </ul>

  <div v-if="showContextMenu" class="context-menu">
    <button>Reply</button>
    <button>Forward</button>
    <button>Delete</button>
  </div>
</template>
```

**Tushuntirish:**
- Pointer Events — mouse, touch, pen'ni bitta API'da birlashtiradi (`mousedown`/`touchstart` alohida handle qilish shart emas)
- `setPointerCapture` — pointer element tashqariga chiqqanda ham event'larni olishni davom ettiradi
- Movement threshold — barmoq titrashi (accidental movement) tufayli long press cancel bo'lmasligi uchun
- Context menu suppress — mobile'da native long press context menu chiqadi, `preventDefault` bilan bloklanadi
- `binding.arg` — `v-longpress:1000` orqali delay ms sifatida uzatiladi

</details>

---

## Challenge 29: Infinite scroll component [Middle+]

**Vazifa:** IntersectionObserver bilan infinite scroll — list oxiriga yetganda yangi data load.

**Talablar:**
- Sentinel element (list oxirida ko'rinmas element) observe qilish
- Loading state va "no more data" holati
- Error handling va retry
- TypeScript generic — har qanday data type bilan ishlasin

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useInfiniteScroll.ts
import {
  ref,
  onMounted,
  onBeforeUnmount,
  type Ref,
} from 'vue'

interface UseInfiniteScrollOptions<T> {
  fetchFn: (page: number) => Promise<T[]>
  pageSize?: number
  rootMargin?: string
}

interface UseInfiniteScrollReturn<T> {
  items: Ref<T[]>
  isLoading: Ref<boolean>
  error: Ref<Error | null>
  hasMore: Ref<boolean>
  sentinelRef: Ref<HTMLElement | null>
  retry: () => void
  reset: () => void
}

export function useInfiniteScroll<T>(
  options: UseInfiniteScrollOptions<T>,
): UseInfiniteScrollReturn<T> {
  const { fetchFn, pageSize = 20, rootMargin = '200px' } = options

  const items = ref<T[]>([]) as Ref<T[]>
  const isLoading = ref(false)
  const error = ref<Error | null>(null)
  const hasMore = ref(true)
  const sentinelRef = ref<HTMLElement | null>(null)

  let page = 1
  let observer: IntersectionObserver | null = null

  async function loadMore() {
    if (isLoading.value || !hasMore.value) return

    isLoading.value = true
    error.value = null

    try {
      const newItems = await fetchFn(page)

      if (newItems.length < pageSize) {
        hasMore.value = false
      }

      items.value = [...items.value, ...newItems]
      page++
    } catch (err) {
      error.value = err instanceof Error ? err : new Error(String(err))
    } finally {
      isLoading.value = false
    }
  }

  function retry() {
    error.value = null
    loadMore()
  }

  function reset() {
    items.value = []
    page = 1
    hasMore.value = true
    error.value = null
    loadMore()
  }

  onMounted(() => {
    observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting && hasMore.value && !isLoading.value) {
          loadMore()
        }
      },
      { rootMargin },
    )

    if (sentinelRef.value) {
      observer.observe(sentinelRef.value)
    }

    // Initial load
    loadMore()
  })

  onBeforeUnmount(() => {
    observer?.disconnect()
    observer = null
  })

  return {
    items,
    isLoading,
    error,
    hasMore,
    sentinelRef,
    retry,
    reset,
  }
}
```

Usage:

```vue
<script setup lang="ts">
import { useInfiniteScroll } from '@/composables/useInfiniteScroll'

interface Article {
  id: number
  title: string
  summary: string
  author: string
}

async function fetchArticles(page: number): Promise<Article[]> {
  const response = await fetch(`/api/articles?page=${page}&limit=20`)
  if (!response.ok) throw new Error('Failed to fetch articles')
  return response.json()
}

const { items: articles, isLoading, error, hasMore, sentinelRef, retry, reset } =
  useInfiniteScroll<Article>({
    fetchFn: fetchArticles,
    pageSize: 20,
    rootMargin: '300px',
  })
</script>

<template>
  <div class="article-feed">
    <button @click="reset">Refresh</button>

    <article v-for="article in articles" :key="article.id" class="article-card">
      <h3>{{ article.title }}</h3>
      <p>{{ article.summary }}</p>
      <span class="author">{{ article.author }}</span>
    </article>

    <!-- Error state -->
    <div v-if="error" class="error-state">
      <p>{{ error.message }}</p>
      <button @click="retry">Retry</button>
    </div>

    <!-- Loading state -->
    <div v-if="isLoading" class="loading">Loading...</div>

    <!-- Sentinel — list oxirida, observe qilinadi -->
    <div
      v-if="hasMore && !error"
      ref="sentinelRef"
      class="sentinel"
      aria-hidden="true"
    />

    <!-- No more data -->
    <p v-if="!hasMore && articles.length > 0" class="end-message">
      All articles loaded
    </p>
  </div>
</template>

<style scoped>
.sentinel { height: 1px; }
.article-card { padding: 1rem; border-bottom: 1px solid #e2e8f0; }
.error-state { padding: 1rem; background: #fee2e2; border-radius: 6px; }
.loading { text-align: center; padding: 1rem; color: #64748b; }
.end-message { text-align: center; padding: 1rem; color: #94a3b8; }
</style>
```

**Tushuntirish:**
- Sentinel element — ko'rinmas `div`, list oxirida. IntersectionObserver sentinel viewport'ga kirganda yangi page load qiladi
- `rootMargin: '300px'` — sentinel hali viewport'ga kirmasdan oldin (300px oldin) trigger bo'ladi — user scroll qilganida data tayyor bo'ladi
- `hasMore` flag — oxirgi page `pageSize`'dan kam item qaytarsa, endi load qilmaslik
- `reset()` — filter/sort o'zgarganda list'ni tozalash va boshidan load qilish

</details>

---

## Challenge 30: Virtual list component (fixed height) [Senior]

**Vazifa:** Minglab item'ni render qilmasdan faqat ko'rinadigan qismini render qiladigan virtual scroll list.

**Talablar:**
- Fixed item height
- Dynamic container height
- Overscan (viewport tashqarisida ham biroz render — smooth scroll uchun)
- TypeScript generic
- Scroll position saqlansin
- Keyboard scroll support (Arrow keys, Page Up/Down)

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- components/VirtualList.vue -->
<script setup lang="ts" generic="T">
import { ref, computed, onMounted, onBeforeUnmount, useTemplateRef } from 'vue'

const props = withDefaults(defineProps<{
  items: T[]
  itemHeight: number
  overscan?: number
  containerHeight?: number
}>(), {
  overscan: 5,
  containerHeight: 400,
})

defineSlots<{
  default(props: { item: T; index: number; style: Record<string, string> }): unknown
}>()

const containerRef = useTemplateRef<HTMLDivElement>('container')
const scrollTop = ref(0)

// Hisoblashlar
const totalHeight = computed(() => props.items.length * props.itemHeight)

const visibleCount = computed(() =>
  Math.ceil(props.containerHeight / props.itemHeight),
)

const startIndex = computed(() => {
  const start = Math.floor(scrollTop.value / props.itemHeight)
  return Math.max(0, start - props.overscan)
})

const endIndex = computed(() => {
  const end = startIndex.value + visibleCount.value + props.overscan * 2
  return Math.min(props.items.length, end)
})

const visibleItems = computed(() => {
  return props.items.slice(startIndex.value, endIndex.value).map((item, i) => ({
    item,
    index: startIndex.value + i,
    style: {
      position: 'absolute' as const,
      top: `${(startIndex.value + i) * props.itemHeight}px`,
      height: `${props.itemHeight}px`,
      width: '100%',
    },
  }))
})

const spacerStyle = computed(() => ({
  height: `${totalHeight.value}px`,
  position: 'relative' as const,
}))

function onScroll() {
  if (containerRef.value) {
    scrollTop.value = containerRef.value.scrollTop
  }
}

function onKeydown(event: KeyboardEvent) {
  const container = containerRef.value
  if (!container) return

  const scrollAmount = {
    ArrowDown: props.itemHeight,
    ArrowUp: -props.itemHeight,
    PageDown: props.containerHeight,
    PageUp: -props.containerHeight,
    Home: -container.scrollTop,
    End: totalHeight.value - container.scrollTop,
  }[event.key]

  if (scrollAmount !== undefined) {
    event.preventDefault()
    container.scrollTop += scrollAmount
  }
}

// Scroll to specific index
function scrollToIndex(index: number, behavior: ScrollBehavior = 'auto') {
  if (containerRef.value) {
    containerRef.value.scrollTo({
      top: index * props.itemHeight,
      behavior,
    })
  }
}

defineExpose({ scrollToIndex })

onMounted(() => {
  containerRef.value?.addEventListener('keydown', onKeydown)
})

onBeforeUnmount(() => {
  containerRef.value?.removeEventListener('keydown', onKeydown)
})
</script>

<template>
  <div
    ref="container"
    class="virtual-list-container"
    :style="{ height: `${containerHeight}px` }"
    @scroll.passive="onScroll"
    tabindex="0"
    role="list"
    :aria-rowcount="items.length"
  >
    <div :style="spacerStyle">
      <slot
        v-for="{ item, index, style } in visibleItems"
        :key="index"
        :item="item"
        :index="index"
        :style="style"
      />
    </div>
  </div>
</template>

<style scoped>
.virtual-list-container {
  overflow-y: auto;
  position: relative;
}

.virtual-list-container:focus {
  outline: 2px solid #3b82f6;
  outline-offset: -2px;
}
</style>
```

Usage:

```vue
<script setup lang="ts">
import { ref, useTemplateRef } from 'vue'
import type { ComponentExposed } from 'vue-component-type-helpers'
import VirtualList from '@/components/VirtualList.vue'

interface LogEntry {
  id: number
  timestamp: string
  level: 'info' | 'warn' | 'error'
  message: string
}

// 100,000 ta log entry
const logEntries = ref<LogEntry[]>(
  Array.from({ length: 100_000 }, (_, i) => ({
    id: i,
    timestamp: new Date(Date.now() - i * 1000).toISOString(),
    level: (['info', 'warn', 'error'] as const)[i % 3],
    message: `Log entry #${i}: ${['Request processed', 'Cache miss', 'Connection timeout'][i % 3]}`,
  })),
)

// Generic komponent uchun InstanceType ishlamaydi — ComponentExposed kerak
const virtualListRef =
  useTemplateRef<ComponentExposed<typeof VirtualList>>('virtualList')

function scrollToLatest() {
  virtualListRef.value?.scrollToIndex(0, 'smooth')
}
</script>

<template>
  <div class="log-viewer">
    <div class="toolbar">
      <span>{{ logEntries.length.toLocaleString() }} entries</span>
      <button @click="scrollToLatest">Scroll to Top</button>
    </div>

    <VirtualList
      ref="virtualList"
      :items="logEntries"
      :item-height="40"
      :overscan="10"
      :container-height="600"
    >
      <template #default="{ item, style }">
        <div
          :style="style"
          class="log-row"
          :class="`log-${item.level}`"
          role="listitem"
        >
          <span class="timestamp">{{ item.timestamp.slice(11, 19) }}</span>
          <span class="level">{{ item.level.toUpperCase() }}</span>
          <span class="message">{{ item.message }}</span>
        </div>
      </template>
    </VirtualList>
  </div>
</template>

<style scoped>
.log-viewer { font-family: monospace; font-size: 0.875rem; }
.toolbar { display: flex; justify-content: space-between; padding: 0.5rem; background: #f1f5f9; }
.log-row { display: flex; gap: 1rem; align-items: center; padding: 0 0.75rem; border-bottom: 1px solid #e2e8f0; }
.timestamp { color: #64748b; width: 70px; flex-shrink: 0; }
.level { width: 50px; font-weight: 600; flex-shrink: 0; }
.message { flex: 1; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.log-info .level { color: #3b82f6; }
.log-warn .level { color: #f59e0b; }
.log-error .level { color: #ef4444; }
</style>
```

**Tushuntirish:**
- Virtual scroll faqat viewport'da ko'rinadigan + overscan item'larni render qiladi — yuqoridagi sozlamada `ceil(600/40) + 10*2 = 35` ta DOM node (100,000 item o'rniga)
- `position: absolute` + `top` — har item'ning aniq pozitsiyasi, scroll paytida recalculate qilinadi
- Spacer `div` — total height'ni beradi (scrollbar to'g'ri ishlashi uchun)
- Overscan — viewport tashqarisida ham bir nechta item render qilinadi (scroll paytida oq bo'shliq ko'rinmasligi uchun)
- `generic="T"` (Vue 3.3+) — component generic type qabul qiladi, slot'da `item` to'g'ri type'ga ega
- `role="list"` + `aria-rowcount` + `role="listitem"` — screen reader'lar uchun accessibility
- Keyboard navigation — Arrow keys, Page Up/Down, Home/End bilan scroll qilish mumkin
- `scrollToIndex` — expose orqali parent'dan specific item'ga scroll qilish

</details>

---

**Vue Interview Yakunlandi.** 7 fayl, 180 ta savol + 30 ta coding challenge. Vue 3.4/3.5+ ekosistemasining barcha asosiy va advanced jihatlari qamrab olingan: Vue Core, Reactivity System, Composition API, Components, Performance, Vue 3.x Features, Coding Challenges.
