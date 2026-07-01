# Bo'lim 6: Form Binding va `v-model`

> `v-model` — two-way binding directive: form input value va component state'ni sinxron saqlaydi. Compiler bu directive'ni `:value` + `@input` (yoki `modelValue` + `update:modelValue` component'lar uchun) kombinatsiyasiga aylantiradi. Vue 3.4+ `defineModel()` macro `v-model` boilerplate'ni keskin kamaytiradi.

---

## Mundarija

- [`v-model` Asoslari](#v-model-asoslari)
- [`v-model` Under the Hood](#v-model-under-the-hood)
- [`v-model` Modifier'lar](#v-model-modifierlar)
- [Form Element'lar bilan `v-model`](#form-elementlar-bilan-v-model)
- [`v-model` Komponent bilan](#v-model-komponent-bilan)
- [`defineModel()` (Vue 3.4+)](#definemodel-vue-34)
- [Multiple `v-model` Bindings](#multiple-v-model-bindings)
- [Custom Modifier'lar](#custom-modifierlar)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## `v-model` Asoslari

### Nazariya

`v-model` — Vue'ning **two-way binding** directive'i. Form input value va component state'ni avtomatik sinxron saqlaydi: input qiymati o'zgarsa — state yangilanadi, state o'zgarsa — input qiymati yangilanadi.

**Sodda misol:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const name = ref('')
</script>

<template>
  <input v-model="name" />
  <p>Hello, {{ name }}!</p>
</template>
```

User input'ga matn yozsa, `name.value` avtomatik yangilanadi. `name.value`'ni script'da o'zgartirsa, input qiymati yangilanadi.

**`v-model` qaerda ishlatish mumkin:**

| Element | Modeled property | Event |
|---------|-----------------|-------|
| `<input type="text">`, `<textarea>` | `value` | `input` |
| `<input type="checkbox">` | `checked` | `change` |
| `<input type="radio">` | `checked` | `change` |
| `<select>` | `value` | `change` |
| Custom component | `modelValue` (prop) | `update:modelValue` (emit) |

**Modeled types:**

| Input type | State type | Misol |
|------------|-----------|-------|
| Text inputs | `string` | `v-model="name"` |
| Number input | `string` (default), `number` (`.number` modifier) | `v-model.number="age"` |
| Checkbox (single) | `boolean` | `v-model="agreed"` |
| Checkbox (multiple) | `string[]` | `v-model="selectedFruits"` |
| Radio | `string` | `v-model="gender"` |
| Select (single) | `string` | `v-model="country"` |
| Select (multiple) | `string[]` | `v-model="hobbies"` |

**Misol — barcha turlar:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const textValue = ref('')
const isAgreed = ref(false)
const selectedFruits = ref<string[]>([])
const gender = ref('')
const country = ref('')
</script>

<template>
  <input v-model="textValue" placeholder="Type..." />

  <label>
    <input type="checkbox" v-model="isAgreed" />
    I agree
  </label>

  <label v-for="fruit in ['Apple', 'Banana', 'Cherry']" :key="fruit">
    <input type="checkbox" :value="fruit" v-model="selectedFruits" />
    {{ fruit }}
  </label>

  <label>
    <input type="radio" value="male" v-model="gender" />
    Male
  </label>
  <label>
    <input type="radio" value="female" v-model="gender" />
    Female
  </label>

  <select v-model="country">
    <option value="">--</option>
    <option value="uz">Uzbekistan</option>
    <option value="us">USA</option>
  </select>
</template>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`v-model` compilation — input uchun:**

Template:
```vue
<input v-model="name" />
```

Compiled:

```javascript
import {
  vModelText as _vModelText,
  withDirectives as _withDirectives,
  createElementVNode as _createElementVNode
} from 'vue'

export function render(_ctx, _cache) {
  return _withDirectives(
    _createElementVNode("input", {
      "onUpdate:modelValue": _cache[0] || (_cache[0] = $event => ((_ctx.name) = $event))
    }, null, 512 /* NEED_PATCH */),
    [[_vModelText, _ctx.name]]
  )
}
```

**Asosiy qismlar:**

1. **`"onUpdate:modelValue"`** — `name` value'ni event qiymati bilan yangilash
2. **`vModelText` directive** — DOM input bilan two-way bind (composition event, IME handling)
3. **`withDirectives`** — directive'ni VNode'ga bog'lash

**`vModelText` runtime directive:**

```typescript
// @vue/runtime-dom/src/directives/vModel.ts (soddalashtirilgan)

// Assigner Symbol kalit ostida saqlanadi (string property emas)
const assignKey: unique symbol = Symbol('_assign')

function castValue(value: string, trim?: boolean, number?: boolean) {
  if (trim) value = value.trim()
  if (number) value = looseToNumber(value)
  return value
}

export const vModelText: ModelDirective<HTMLInputElement | HTMLTextAreaElement> = {
  created(el, { modifiers: { lazy, trim, number } }, vnode) {
    el[assignKey] = getModelAssigner(vnode)
    const castToNumber = number || (vnode.props && vnode.props.type === 'number')

    addEventListener(el, lazy ? 'change' : 'input', (e) => {
      if ((e.target as any).composing) return  // IME composition
      el[assignKey](castValue(el.value, trim, castToNumber))
    })

    if (trim) {
      addEventListener(el, 'change', () => {
        el.value = castValue(el.value, trim, castToNumber)
      })
    }

    if (!lazy) {
      addEventListener(el, 'compositionstart', onCompositionStart)
      addEventListener(el, 'compositionend', onCompositionEnd)
      addEventListener(el, 'change', onCompositionEnd)
    }
  },
  mounted(el, { value }) {
    el.value = value == null ? '' : value
  },
  beforeUpdate(el, { value, modifiers: { lazy, trim, number } }, vnode) {
    el[assignKey] = getModelAssigner(vnode)
    if ((el as any).composing) return
    const elValue = number || el.type === 'number' ? looseToNumber(el.value) : el.value
    const newValue = value == null ? '' : value
    if (elValue !== newValue) {
      if (el.ownerDocument.activeElement === el && el.type !== 'range') {
        if (lazy) return
        if (trim && el.value.trim() === newValue) return
      }
      el.value = newValue
    }
  }
}
```

**Asosiy nuans'lar:**

1. **IME (Input Method Editor)** — Chinese, Japanese, Korean characters typing. `compositionstart`/`compositionend` event'lar — composition mode'da DOM update'ni to'xtatish, til input tugagandan keyin update
2. **Active element check** — input focused bo'lsa, value'ni qayta o'rnatish (cursor o'rni saqlash) — careful

**Boshqa `vModel*` directive'lar:**

| Directive | Element |
|-----------|---------|
| `vModelText` | `<input type="text">`, `<textarea>` |
| `vModelCheckbox` | `<input type="checkbox">` |
| `vModelRadio` | `<input type="radio">` |
| `vModelSelect` | `<select>` |
| `vModelDynamic` | Dynamic `<input :type="...">` |

Manba: [Vue.js `v-model`](https://vuejs.org/guide/essentials/forms.html), [`@vue/runtime-dom` vModel source](https://github.com/vuejs/core/blob/main/packages/runtime-dom/src/directives/vModel.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Login form — to'liq misol:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface LoginCredentials {
  email: string
  password: string
  rememberMe: boolean
}

const credentials = ref<LoginCredentials>({
  email: '',
  password: '',
  rememberMe: false
})

const isValid = computed(() => {
  return (
    /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(credentials.value.email) &&
    credentials.value.password.length >= 8
  )
})

async function login() {
  if (!isValid.value) return
  // ... API call
}
</script>

<template>
  <form @submit.prevent="login">
    <input
      v-model="credentials.email"
      type="email"
      placeholder="Email"
      required
    />
    <input
      v-model="credentials.password"
      type="password"
      placeholder="Password (min 8 chars)"
      required
    />
    <label>
      <input type="checkbox" v-model="credentials.rememberMe" />
      Remember me
    </label>
    <button type="submit" :disabled="!isValid">Login</button>
  </form>
</template>
```

**Multi-checkbox — array binding:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const selectedTags = ref<string[]>([])
const allTags = ['vue', 'react', 'angular', 'svelte', 'solid']
</script>

<template>
  <fieldset>
    <legend>Select frameworks:</legend>
    <label v-for="tag in allTags" :key="tag">
      <input type="checkbox" :value="tag" v-model="selectedTags" />
      {{ tag }}
    </label>
  </fieldset>
  <p>Selected: {{ selectedTags.join(', ') || 'none' }}</p>
</template>
```

</details>

---

## `v-model` Under the Hood

### Nazariya

`v-model` — syntactic sugar. Underlying mexanizm:

**Native input uchun:**

```vue
<!-- v-model -->
<input v-model="text" />

<!-- Ekvivalent -->
<input :value="text" @input="(e) => text = (e.target as HTMLInputElement).value" />
```

**Component uchun (Vue 3 default):**

```vue
<!-- v-model -->
<MyInput v-model="text" />

<!-- Ekvivalent -->
<MyInput :model-value="text" @update:model-value="(value) => text = value" />
```

**Component implementation:**

```vue
<!-- MyInput.vue (Vue 3.3-, eski pattern) -->
<script setup lang="ts">
const props = defineProps<{ modelValue: string }>()
const emit = defineEmits<{ 'update:modelValue': [value: string] }>()
</script>

<template>
  <input
    :value="modelValue"
    @input="emit('update:modelValue', ($event.target as HTMLInputElement).value)"
  />
</template>
```

Vue 3.4+ `defineModel()` macro bilan boilerplate kamayadi (pastda batafsil).

**Vue 2 vs Vue 3 farqi:**

| Vue 2 | Vue 3 |
|-------|-------|
| Prop: `value` | Prop: `modelValue` |
| Event: `input` | Event: `update:modelValue` |
| `model` option bilan customize | Multiple `v-model` native support |

**Vue 3 yangi:**

- Multiple `v-model`: `v-model:title`, `v-model:content` — bir component'da bir nechta
- Argument bilan: `v-model:propName="value"` → prop=`propName`, event=`update:propName`
- Custom modifier qo'llab-quvvatlash

<details>
<summary><strong>Under the Hood</strong></summary>

**Component `v-model` compilation:**

Template:
```vue
<MyInput v-model="text" />
```

Compiled:

```javascript
import { createVNode as _createVNode } from 'vue'

return _createVNode(MyInput, {
  modelValue: _ctx.text,
  "onUpdate:modelValue": _cache[0] || (_cache[0] = $event => ((_ctx.text) = $event))
})
```

**Argument bilan:**

```vue
<MyComp v-model:title="title" />
```

Compiled:

```javascript
return _createVNode(MyComp, {
  title: _ctx.title,
  "onUpdate:title": _cache[0] || (_cache[0] = $event => ((_ctx.title) = $event))
})
```

**Multiple v-model:**

```vue
<UserForm v-model:name="userName" v-model:email="userEmail" />
```

Compiled:

```javascript
return _createVNode(UserForm, {
  name: _ctx.userName,
  "onUpdate:name": _cache[0] || (_cache[0] = $event => ((_ctx.userName) = $event)),
  email: _ctx.userEmail,
  "onUpdate:email": _cache[1] || (_cache[1] = $event => ((_ctx.userEmail) = $event))
})
```

**Cache pattern:** `_cache[0]` — har render'da yangi function emas, bir marta yaratiladi.

Manba: [Vue.js Component `v-model`](https://vuejs.org/guide/components/v-model.html)

</details>

---

## `v-model` Modifier'lar

### Nazariya

Vue uch ta built-in modifier qo'llab-quvvatlaydi:

| Modifier | Vazifa |
|----------|--------|
| **`.lazy`** | `input` event o'rniga `change` event (focus tashlaganda yoki Enter bilan) |
| **`.number`** | String'ni number'ga avtomatik cast |
| **`.trim`** | Boshi-oxiri whitespace olib tashlash |

**Misollar:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const lazyValue = ref('')
const numberValue = ref(0)
const trimmedValue = ref('')
</script>

<template>
  <!-- .lazy — har keystroke'da emas, change event'da update -->
  <input v-model.lazy="lazyValue" />

  <!-- .number — string → number cast -->
  <input v-model.number="numberValue" type="number" />
  <p>Type: {{ typeof numberValue }}</p>

  <!-- .trim — whitespace o'chirish -->
  <input v-model.trim="trimmedValue" />
</template>
```

**Modifier'lar chaining:**

```vue
<!-- Trim + number -->
<input v-model.trim.number="value" />

<!-- Lazy + trim -->
<input v-model.lazy.trim="value" />
```

**`.lazy` qachon foydali:**

- Heavy computation har keystroke'da kerak emas (mas. autocomplete debouncing emas, balki "Enter" bilan submit)
- Validation focus tashlangach (`.lazy` + `@blur` ekvivalent)

**`.number` nuance:**

DOM input value har doim String. `.number` modifier value'ni `looseToNumber()` orqali number'ga cast qiladi. Muhim nuance: `<input type="number">` bo'lganda `.number` **avtomatik qo'llaniladi** — `vModelText` directive ichida `castToNumber = number || vnode.props.type === 'number'`, ya'ni `type="number"` o'zi cast'ni yoqadi, modifier yozish shart emas:

```vue
<!-- type="number" — .number avtomatik qo'llaniladi -->
<input v-model="age" type="number" />
<!-- age = 25 (number) -->

<!-- type="text" — String qoladi, cast yo'q -->
<input v-model="age" />
<!-- age = "25" (string) -->

<!-- .number ni explicit yozish — type="text" uchun cast yoqadi -->
<input v-model.number="age" />
<!-- age = 25 (number) -->
```

`looseToNumber()` — Vue cast utility: agar parse muvaffaqiyatli bo'lsa number, aks holda original string (`""` empty string `NaN` check tufayli `""` qoladi).

**`.trim` muhimligi:**

```vue
<!-- User: "  Ali  " yozdi (boshi-oxiri space) -->

<!-- ❌ Trim'siz -->
<input v-model="name" />
<!-- name = "  Ali  " — search/login'da bug -->

<!-- ✅ Trim bilan -->
<input v-model.trim="name" />
<!-- name = "Ali" -->
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Modifier'lar `vModelText` directive ichida:**

```typescript
// @vue/runtime-dom/src/directives/vModel.ts (soddalashtirilgan)
export const vModelText: ModelDirective = {
  created(el, { modifiers: { lazy, trim, number } }, vnode) {
    el[assignKey] = getModelAssigner(vnode)
    const castToNumber = number || (vnode.props && vnode.props.type === 'number')

    addEventListener(el, lazy ? 'change' : 'input', (e) => {
      if ((e.target as any).composing) return  // IME
      // castValue: trim → number tartibida transform
      el[assignKey](castValue(el.value, trim, castToNumber))
    })

    if (trim) {
      addEventListener(el, 'change', () => {
        el.value = castValue(el.value, trim, castToNumber)
      })
    }
  }
}
```

**`looseToNumber()` utility:**

```typescript
// @vue/shared/src/general.ts
export const looseToNumber = (val: any): any => {
  const n = parseFloat(val)
  return isNaN(n) ? val : n
}
```

Agar parse muvaffaqiyatli — number qaytaradi, aks holda original. Bu sabab `.number` modifier bilan empty string `""` empty string qoladi (NaN check).

**`.lazy` mexanizm:**

- Default: `input` event — har keystroke'da
- `.lazy`: `change` event — focus tashlash yoki Enter (input element default behavior)

Bu `addEventListener(el, 'input' yoki 'change', ...)` orqali implement qilingan.

Manba: [Vue.js Form Modifiers](https://vuejs.org/guide/essentials/forms.html#modifiers)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Search debounce — lazy alternative:**

```vue
<script setup lang="ts">
import { ref, watch } from 'vue'
import { useDebounceFn } from '@vueuse/core'

const query = ref('')
const results = ref<string[]>([])

const debouncedSearch = useDebounceFn(async (q: string) => {
  if (!q.trim()) {
    results.value = []
    return
  }
  // API call
}, 300)

watch(query, debouncedSearch)
</script>

<template>
  <!-- v-model.lazy — submit'da search, real-time emas -->
  <input v-model.lazy="query" placeholder="Press Enter to search..." />

  <!-- yoki: real-time search with debounce -->
  <!-- <input v-model="query" placeholder="Search..." /> -->

  <ul>
    <li v-for="r in results" :key="r">{{ r }}</li>
  </ul>
</template>
```

**Currency input — number + trim:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const priceInput = ref(0)

const formatted = computed(() => priceInput.value.toLocaleString('uz-UZ'))
</script>

<template>
  <label>
    Price:
    <input v-model.number.trim="priceInput" type="number" min="0" step="0.01" />
  </label>
  <p>Formatted: {{ formatted }} so'm</p>
</template>
```

**Username — trim majburiy:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const username = ref('')
const isValid = computed(() => /^[a-zA-Z0-9_]{3,20}$/.test(username.value))
</script>

<template>
  <input
    v-model.trim="username"
    placeholder="Username (3-20 chars, alphanumeric)"
    :class="{ invalid: username && !isValid }"
  />
  <p v-if="username && !isValid" class="error">Invalid username</p>
</template>
```

`.trim` — user space yozsa ham, validation to'g'ri ishlaydi.

</details>

---

## Form Element'lar bilan `v-model`

### Nazariya

Har form element turi `v-model` bilan turlicha ishlaydi. Vue avtomatik to'g'ri directive'ni tanlaydi (`vModelText`, `vModelCheckbox`, `vModelRadio`, `vModelSelect`).

**1. Text input va textarea:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const text = ref('')
const description = ref('')
</script>

<template>
  <input v-model="text" type="text" />
  <textarea v-model="description"></textarea>
</template>
```

**❌ Textarea ichida interpolation — taqiq:**

```vue
<!-- ❌ Bu ishlamaydi -->
<textarea>{{ description }}</textarea>

<!-- ✅ v-model bilan -->
<textarea v-model="description"></textarea>
```

**2. Checkbox — single (boolean):**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const isAgreed = ref(false)
</script>

<template>
  <input type="checkbox" v-model="isAgreed" />
  <p>Agreed: {{ isAgreed }}</p>
</template>
```

**Custom value (true/false o'rniga):**

```vue
<input type="checkbox" v-model="toggle" true-value="yes" false-value="no" />
<!-- checked → toggle = "yes", unchecked → toggle = "no" -->
```

**3. Checkbox — multiple (array):**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const selectedFruits = ref<string[]>([])
</script>

<template>
  <input type="checkbox" value="apple" v-model="selectedFruits" />
  <input type="checkbox" value="banana" v-model="selectedFruits" />
  <input type="checkbox" value="cherry" v-model="selectedFruits" />
  <!-- selectedFruits = ['apple', 'cherry'] (tanlanganlar) -->
</template>
```

**4. Radio:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const gender = ref('')
</script>

<template>
  <input type="radio" value="male" v-model="gender" />
  <input type="radio" value="female" v-model="gender" />
  <input type="radio" value="other" v-model="gender" />
  <!-- gender = "male" yoki "female" yoki "other" -->
</template>
```

**5. Select — single:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const country = ref('')
</script>

<template>
  <select v-model="country">
    <option disabled value="">Please select</option>
    <option value="uz">Uzbekistan</option>
    <option value="us">USA</option>
    <option value="ru">Russia</option>
  </select>
</template>
```

**6. Select — multiple:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const hobbies = ref<string[]>([])
</script>

<template>
  <select v-model="hobbies" multiple>
    <option value="reading">Reading</option>
    <option value="coding">Coding</option>
    <option value="gaming">Gaming</option>
  </select>
  <!-- hobbies = ['reading', 'coding'] -->
</template>
```

**7. Dynamic options (`v-for`):**

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface SelectOption {
  value: string
  label: string
}

const options = ref<SelectOption[]>([
  { value: 'a', label: 'Option A' },
  { value: 'b', label: 'Option B' }
])
const selected = ref('')
</script>

<template>
  <select v-model="selected">
    <option v-for="opt in options" :key="opt.value" :value="opt.value">
      {{ opt.label }}
    </option>
  </select>
</template>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`vModelCheckbox` — array vs boolean detection:**

```typescript
// @vue/runtime-dom/src/directives/vModel.ts
export const vModelCheckbox: ModelDirective = {
  // array checkbox value'lari deep traverse qilinadi
  deep: true,
  created(el, _, vnode) {
    el[assignKey] = getModelAssigner(vnode)
    addEventListener(el, 'change', () => {
      const modelValue = (el as any)._modelValue
      const elementValue = getValue(el)
      const checked = el.checked
      const assign = el[assignKey]

      if (isArray(modelValue)) {
        const index = looseIndexOf(modelValue, elementValue)
        const found = index !== -1
        if (checked && !found) {
          assign(modelValue.concat(elementValue))
        } else if (!checked && found) {
          const filtered = [...modelValue]
          filtered.splice(index, 1)
          assign(filtered)
        }
      } else if (isSet(modelValue)) {
        // Set bilan ham ishlaydi
      } else {
        assign(getCheckboxValue(el, checked))  // boolean yoki custom value
      }
    })
  },
  mounted: setChecked,
  beforeUpdate: setChecked
}
```

**Algorithm:**

1. `modelValue` array → checkbox bo'lsa add, unchecked bo'lsa remove
2. `modelValue` Set → array bilan o'xshash
3. `modelValue` boolean/value → checked state assign

**`vModelSelect` — multiple option:**

```typescript
export const vModelSelect: ModelDirective = {
  // deep watch — multi-select array/Set ichidagi o'zgarishni ham kuzatadi
  deep: true,
  created(el, { value, modifiers: { number } }, vnode) {
    const isSetModel = isSet(value)
    addEventListener(el, 'change', () => {
      const selectedVal = Array.prototype.filter
        .call(el.options, (o: HTMLOptionElement) => o.selected)
        .map((o: HTMLOptionElement) =>
          number ? looseToNumber(getValue(o)) : getValue(o)
        )
      el[assignKey](
        el.multiple
          ? isSetModel ? new Set(selectedVal) : selectedVal
          : selectedVal[0]
      )
    })
    el[assignKey] = getModelAssigner(vnode)
  },
  mounted(el, { value }) {
    setSelected(el, value)
  },
  beforeUpdate(el, _binding, vnode) {
    el[assignKey] = getModelAssigner(vnode)
  },
  updated(el, { value }) {
    setSelected(el, value)
  }
}
```

**Dynamic input type (`<input :type="...">`)** — `vModelDynamic`:

Runtime'da type'ni tekshirib mos directive tanlanadi.

Manba: [Vue.js Form Binding source](https://github.com/vuejs/core/blob/main/packages/runtime-dom/src/directives/vModel.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Survey form — barcha element turlari:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface SurveyData {
  name: string
  email: string
  age: number
  gender: 'male' | 'female' | 'other' | ''
  interests: string[]
  experience: string
  agreeTerms: boolean
  newsletter: boolean
  feedback: string
}

const survey = ref<SurveyData>({
  name: '',
  email: '',
  age: 0,
  gender: '',
  interests: [],
  experience: '',
  agreeTerms: false,
  newsletter: false,
  feedback: ''
})

function submit() {
  console.log('Survey data:', survey.value)
}
</script>

<template>
  <form @submit.prevent="submit">
    <input v-model.trim="survey.name" placeholder="Name" required />
    <input v-model.trim="survey.email" type="email" placeholder="Email" required />
    <input v-model.number="survey.age" type="number" placeholder="Age" min="0" />

    <fieldset>
      <legend>Gender</legend>
      <label><input type="radio" value="male" v-model="survey.gender" /> Male</label>
      <label><input type="radio" value="female" v-model="survey.gender" /> Female</label>
      <label><input type="radio" value="other" v-model="survey.gender" /> Other</label>
    </fieldset>

    <fieldset>
      <legend>Interests (multiple)</legend>
      <label v-for="i in ['Tech', 'Sports', 'Music', 'Art']" :key="i">
        <input type="checkbox" :value="i" v-model="survey.interests" />
        {{ i }}
      </label>
    </fieldset>

    <select v-model="survey.experience">
      <option value="">Select experience level</option>
      <option value="junior">Junior (0-2 years)</option>
      <option value="middle">Middle (2-5 years)</option>
      <option value="senior">Senior (5+ years)</option>
    </select>

    <textarea v-model.trim="survey.feedback" rows="4" placeholder="Feedback..."></textarea>

    <label>
      <input type="checkbox" v-model="survey.agreeTerms" required />
      I agree to terms and conditions
    </label>
    <label>
      <input type="checkbox" v-model="survey.newsletter" />
      Subscribe to newsletter
    </label>

    <button type="submit">Submit</button>
  </form>
</template>
```

</details>

---

## `v-model` Komponent bilan

### Nazariya

Custom component'ga `v-model` qo'shish — wrapper component yaratish (mas. `CustomInput`, `DatePicker`, `ColorPicker`).

**Vue 3.3- (eski pattern) — `defineProps` + `defineEmits`:**

```vue
<!-- CustomInput.vue -->
<script setup lang="ts">
const props = defineProps<{ modelValue: string }>()
const emit = defineEmits<{ 'update:modelValue': [value: string] }>()
</script>

<template>
  <input
    :value="modelValue"
    @input="emit('update:modelValue', ($event.target as HTMLInputElement).value)"
  />
</template>
```

**Parent ishlatish:**

```vue
<CustomInput v-model="text" />
```

**Vue 3.4+ — `defineModel()` macro (tavsiya etilgan):**

```vue
<!-- CustomInput.vue (3.4+) -->
<script setup lang="ts">
const model = defineModel<string>()
</script>

<template>
  <input v-model="model" />
</template>
```

Hech qanday explicit `defineProps`, `defineEmits` yo'q — macro avtomatik handle qiladi.

**Argument bilan (named `v-model`):**

```vue
<!-- UserForm.vue -->
<script setup lang="ts">
const name = defineModel<string>('name', { default: '' })
const email = defineModel<string>('email', { default: '' })
</script>

<template>
  <input v-model="name" placeholder="Name" />
  <input v-model="email" placeholder="Email" />
</template>
```

Parent:

```vue
<UserForm v-model:name="userName" v-model:email="userEmail" />
```

**`required` va `default` bilan:**

```vue
<script setup lang="ts">
const name = defineModel<string>('name', { required: true })
const age = defineModel<number>('age', { default: 0 })
</script>
```

**Chuqurroq:** [21-script-setup-advanced.md](21-script-setup-advanced.md)

<details>
<summary><strong>Under the Hood</strong></summary>

**Eski pattern compilation:**

```vue
<MyInput v-model="text" />
```

Compiled:

```javascript
return _createVNode(MyInput, {
  modelValue: _ctx.text,
  "onUpdate:modelValue": $event => _ctx.text = $event
})
```

Component bu props/event'larni o'zi handle qiladi (`defineProps`, `defineEmits` orqali).

**`defineModel()` compilation (Vue 3.4+):**

Source:

```vue
<script setup lang="ts">
const model = defineModel<string>()
</script>
```

Compiled (taxminiy):

```javascript
import { useModel } from 'vue'

export default {
  emits: ['update:modelValue'],
  props: {
    modelValue: {}
  },
  setup(__props, { expose }) {
    const model = useModel(__props, 'modelValue')
    return { model }
  }
}
```

**`useModel()` ichida:**

`useModel` `computed` qaytarmaydi — `customRef`-ga o'xshash mexanizm bilan **local value buffer** saqlaydi. Local buffer kerak: setter emit chaqirgandan keyin parent state'ni darhol yangilamasligi mumkin (parent `v-model` ulamagan, yoki async update), shu paytda component o'z value'sini ko'rsata olishi shart.

```typescript
// @vue/runtime-core/src/helpers/useModel.ts (soddalashtirilgan)
export function useModel(props: Record<string, any>, name: string): Ref {
  const i = getCurrentInstance()!
  let localValue: any

  // prop o'zgarsa local buffer'ni sinxronlaydi
  watchSyncEffect(() => {
    const propValue = props[name]
    if (hasChanged(localValue, propValue)) {
      localValue = propValue
    }
  })

  const res = customRef((track, trigger) => ({
    get() {
      track()
      return localValue
    },
    set(value) {
      if (hasChanged(value, localValue)) {
        localValue = value
        trigger()
      }
      // hyphenate/camelize — ikkala event nomini ham hisobga oladi
      i.emit(`update:${name}`, value)
    }
  }))

  return res
}
```

`defineModel()` reactive `Ref` qaytaradi:
- Read: local buffer'dan o'qiydi (prop bilan `watchSyncEffect` orqali sinxron)
- Write: local buffer'ni yangilaydi va `update:propName` event emit qiladi → parent state yangilanadi (ulangan bo'lsa)

**Local mutation behavior:**

```vue
<script setup lang="ts">
const model = defineModel<string>()

// .value = ... — local buffer'ni yangilaydi va emit qiladi
model.value = 'new'  // local buffer = 'new' + emit('update:modelValue', 'new')
</script>
```

Parent `v-model` ulamasa ham, `model.value = 'new'` local buffer'ni yangilagani uchun component yangi value'ni darhol ko'radi. Emit baribir chaqiriladi, lekin parent listen qilmasa parent state o'zgarmaydi — component mustaqil state sifatida ishlaydi.

Manba: [Vue 3.4 `defineModel`](https://blog.vuejs.org/posts/vue-3-4#definemodel-stabilization), [`useModel` source](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/helpers/useModel.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Eski pattern — CurrencyInput (3.3-):**

```vue
<!-- CurrencyInput.vue -->
<script setup lang="ts">
import { computed } from 'vue'

interface Props {
  modelValue: number
  currency?: string
  min?: number
  max?: number
}

const props = withDefaults(defineProps<Props>(), {
  currency: '$',
  min: 0,
  max: Number.MAX_SAFE_INTEGER
})

const emit = defineEmits<{ 'update:modelValue': [value: number] }>()

const displayValue = computed({
  get: () => props.modelValue.toFixed(2),
  set: (val: string) => {
    const num = parseFloat(val)
    if (!isNaN(num) && num >= props.min && num <= props.max) {
      emit('update:modelValue', num)
    }
  }
})
</script>

<template>
  <div class="currency-input">
    <span class="symbol">{{ currency }}</span>
    <input v-model="displayValue" type="text" />
  </div>
</template>
```

Parent:

```vue
<script setup lang="ts">
import { ref } from 'vue'
const price = ref(100.00)
</script>

<template>
  <CurrencyInput v-model="price" currency="$" :min="0" :max="10000" />
  <p>Price: {{ price }}</p>
</template>
```

**`defineModel()` bilan (3.4+):**

```vue
<!-- CurrencyInput.vue -->
<script setup lang="ts">
import { computed } from 'vue'

interface Props {
  currency?: string
}

withDefaults(defineProps<Props>(), { currency: '$' })

// v-model — kompakt
const price = defineModel<number>({ required: true })

const displayValue = computed({
  get: () => price.value.toFixed(2),
  set: (val: string) => {
    const num = parseFloat(val)
    if (!isNaN(num)) price.value = num
  }
})
</script>

<template>
  <div class="currency-input">
    <span>{{ currency }}</span>
    <input v-model="displayValue" type="text" />
  </div>
</template>
```

**Boilerplate farqi:**
- Eski: 4 qator (props, emits, getter, setter)
- Yangi: 1 qator (`defineModel()`)

</details>

---

## `defineModel()` (Vue 3.4+)

### Nazariya

`defineModel()` — Vue 3.4'da stable bo'lgan compiler macro. `v-model` boilerplate'ni keskin kamaytiradi.

**Asosiy syntax:**

```vue
<script setup lang="ts">
// Default modelValue
const model = defineModel()

// TypeScript bilan
const text = defineModel<string>()

// Argument (named v-model)
const title = defineModel<string>('title')

// Options bilan
const count = defineModel<number>({ default: 0, required: true })
</script>
```

**Avval (Vue 3.3-) vs Hozir (Vue 3.4+) taqqoslash:**

```vue
<!-- ❌ Vue 3.3- (10+ qator boilerplate) -->
<script setup lang="ts">
const props = defineProps<{
  modelValue: string
}>()

const emit = defineEmits<{
  'update:modelValue': [value: string]
}>()

function update(e: Event) {
  emit('update:modelValue', (e.target as HTMLInputElement).value)
}
</script>

<template>
  <input :value="modelValue" @input="update" />
</template>
```

```vue
<!-- ✅ Vue 3.4+ (3 qator) -->
<script setup lang="ts">
const model = defineModel<string>()
</script>

<template>
  <input v-model="model" />
</template>
```

**`defineModel()` return type:**

```typescript
const model = defineModel<string>()
// Type: Ref<string | undefined>

const model = defineModel<string>({ required: true })
// Type: Ref<string>

const model = defineModel<string>({ default: 'hello' })
// Type: Ref<string>
```

**Reactivity:**

- `model.value` — get parent state
- `model.value = newVal` — emit update event (parent state yangilanadi)
- Template'da `v-model` to'g'ridan-to'g'ri ishlaydi

**Parent `v-model` bo'lmasa:**

`defineModel()` parent `v-model` bermasa ham ishlaydi — local `Ref` sifatida. Value o'zgarsa emit chaqiriladi, lekin parent listen qilmaydi. Bu component'ni mustaqil state bilan ham ishlatish imkonini beradi:

```vue
<script setup lang="ts">
const model = defineModel<string>({ default: '' })
// Parent v-model bersa — sync, bermasa — local state sifatida ishlaydi
</script>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Compiler transform — `defineModel()` qanday boshqacha qilinadi:**

Source:

```vue
<script setup lang="ts">
const model = defineModel<string>()
const title = defineModel<string>('title', { default: 'untitled' })
</script>
```

Compiled (soddalashtirilgan):

```javascript
import { useModel } from 'vue'

export default {
  emits: ['update:modelValue', 'update:title'],
  props: {
    modelValue: {},
    title: { default: 'untitled' }
  },
  setup(__props) {
    const model = useModel(__props, 'modelValue')
    const title = useModel(__props, 'title')
    return { model, title }
  }
}
```

**Compiler avtomatik qiladi:**

1. **`emits` ro'yxati** — har `defineModel()` uchun `update:propName` event qo'shadi
2. **`props` ro'yxati** — har `defineModel()` uchun prop e'lon qiladi
3. **`useModel` runtime helper** — reactive Ref qaytaradi

**TypeScript type inference:**

```typescript
declare function defineModel<T = any>(): Ref<T | undefined>
declare function defineModel<T>(options: { required: true }): Ref<T>
declare function defineModel<T>(options: { default: T }): Ref<T>
declare function defineModel<T>(name: string): Ref<T | undefined>
declare function defineModel<T>(name: string, options: { required?: boolean; default?: T }): Ref<T>
```

Manba: [Vue 3.4 release notes](https://blog.vuejs.org/posts/vue-3-4), [`useModel` source](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/helpers/useModel.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**SearchBar component:**

```vue
<!-- SearchBar.vue -->
<script setup lang="ts">
const query = defineModel<string>({ default: '' })

// placeholder — one-way prop, defineModel kerak emas
const props = withDefaults(defineProps<{ placeholder?: string }>(), {
  placeholder: 'Search...'
})

function clear() {
  query.value = ''
}
</script>

<template>
  <div class="search-bar">
    <input
      v-model="query"
      :placeholder="props.placeholder"
      type="search"
    />
    <button v-if="query" @click="clear">×</button>
  </div>
</template>
```

Parent:

```vue
<script setup lang="ts">
import { ref } from 'vue'
import SearchBar from './SearchBar.vue'

const search = ref('')
</script>

<template>
  <SearchBar v-model="search" placeholder="Type to search..." />
  <p>Searching: {{ search }}</p>
</template>
```

**DatePicker — typed defineModel:**

```vue
<!-- DatePicker.vue -->
<script setup lang="ts">
import { computed } from 'vue'

const date = defineModel<Date>({ required: true })

const formattedDate = computed({
  get: () => date.value.toISOString().slice(0, 10),  // YYYY-MM-DD
  set: (val: string) => {
    const newDate = new Date(val)
    if (!isNaN(newDate.getTime())) {
      date.value = newDate
    }
  }
})
</script>

<template>
  <input v-model="formattedDate" type="date" />
</template>
```

**Required + validation:**

```vue
<!-- AgeInput.vue -->
<script setup lang="ts">
import { computed } from 'vue'

const age = defineModel<number>({ required: true })

const isValid = computed(() => age.value >= 0 && age.value <= 150)
</script>

<template>
  <div class="age-input">
    <input
      v-model.number="age"
      type="number"
      min="0"
      max="150"
      :class="{ invalid: !isValid }"
    />
    <span v-if="!isValid" class="error">Invalid age</span>
  </div>
</template>
```

</details>

---

## Multiple `v-model` Bindings

### Nazariya

Vue 3'da bir component'da bir nechta `v-model` ishlatish mumkin — har biri unique nom bilan (`v-model:propName`):

```vue
<!-- AddressInput.vue -->
<script setup lang="ts">
// Vue 3.4+
const street = defineModel<string>('street', { default: '' })
const city = defineModel<string>('city', { default: '' })
const zip = defineModel<string>('zip', { default: '' })
</script>

<template>
  <fieldset>
    <input v-model="street" placeholder="Street" />
    <input v-model="city" placeholder="City" />
    <input v-model="zip" placeholder="ZIP" />
  </fieldset>
</template>
```

Parent:

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface Address {
  street: string
  city: string
  zip: string
}

const address = ref<Address>({
  street: '',
  city: '',
  zip: ''
})
</script>

<template>
  <AddressInput
    v-model:street="address.street"
    v-model:city="address.city"
    v-model:zip="address.zip"
  />
</template>
```

**Vue 3.3- eski pattern (bir nechta v-model):**

```vue
<!-- AddressInput.vue (3.3-) -->
<script setup lang="ts">
defineProps<{
  street: string
  city: string
  zip: string
}>()

const emit = defineEmits<{
  'update:street': [value: string]
  'update:city': [value: string]
  'update:zip': [value: string]
}>()
</script>

<template>
  <input :value="street" @input="emit('update:street', ($event.target as HTMLInputElement).value)" />
  <!-- ... -->
</template>
```

**Asosiy syntax taqqoslash:**

| Single `v-model` | Multiple `v-model` |
|------------------|---------------------|
| `v-model="value"` | `v-model:title="title" v-model:content="content"` |
| Prop: `modelValue` | Prop: `title`, `content` |
| Event: `update:modelValue` | Event: `update:title`, `update:content` |

**Real-world use case:** Range slider (`min` + `max`), date range (`from` + `to`), form section (multi-field):

```vue
<!-- DateRangePicker.vue -->
<script setup lang="ts">
// date input string formatda ishlaydi (YYYY-MM-DD)
const from = defineModel<string>('from', { default: '' })
const to = defineModel<string>('to', { default: '' })
</script>

<template>
  <input v-model="from" type="date" />
  <input v-model="to" type="date" />
</template>
```

Parent:

```vue
<DateRangePicker v-model:from="startDate" v-model:to="endDate" />
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Compilation — multiple v-model:**

Template:

```vue
<UserForm v-model:name="userName" v-model:email="userEmail" />
```

Compiled:

```javascript
return _createVNode(UserForm, {
  name: _ctx.userName,
  "onUpdate:name": _cache[0] || (_cache[0] = $event => ((_ctx.userName) = $event)),
  email: _ctx.userEmail,
  "onUpdate:email": _cache[1] || (_cache[1] = $event => ((_ctx.userEmail) = $event))
})
```

**Component side (`defineModel` bilan):**

```vue
<script setup lang="ts">
const name = defineModel<string>('name')
const email = defineModel<string>('email')
</script>
```

Compiled:

```javascript
export default {
  emits: ['update:name', 'update:email'],
  props: {
    name: {},
    email: {}
  },
  setup(__props) {
    const name = useModel(__props, 'name')
    const email = useModel(__props, 'email')
    return { name, email }
  }
}
```

Har `defineModel(arg)` chaqirig'i alohida prop + event qo'shadi.

Manba: [Vue.js Multiple v-model](https://vuejs.org/guide/components/v-model.html#multiple-v-model-bindings)

</details>

---

## Custom Modifier'lar

### Nazariya

Vue built-in modifier'lardan tashqari (`lazy`, `trim`, `number`), component custom modifier'larni qo'llab-quvvatlashi mumkin.

**Vue 3.4+ — `defineModel()` modifiers:**

```vue
<!-- CustomInput.vue -->
<script setup lang="ts">
const [model, modifiers] = defineModel<string>({
  set(value) {
    if (modifiers.capitalize) {
      return value.charAt(0).toUpperCase() + value.slice(1)
    }
    return value
  }
})
</script>

<template>
  <input v-model="model" />
</template>
```

Parent — custom `.capitalize` modifier:

```vue
<CustomInput v-model.capitalize="text" />
```

User "hello" yozsa, `text = "Hello"` (avto-capitalize).

**`defineModel()` tuple return:**

```typescript
const [model, modifiers] = defineModel<string>({ ... })
// model: Ref<string>
// modifiers: { [key: string]: boolean }  — { capitalize: true } agar v-model.capitalize ishlatilgan bo'lsa
```

**`set` hook** — value parent'ga emit qilinishidan oldin transform:

```typescript
defineModel({
  set(value) {
    // Transform value
    return transformedValue
  }
})
```

**Vue 3.3- eski pattern:**

```vue
<!-- 3.3- -->
<script setup lang="ts">
const props = defineProps<{
  modelValue: string
  modelModifiers?: Record<string, boolean>
}>()

const emit = defineEmits<{ 'update:modelValue': [value: string] }>()

function update(value: string) {
  if (props.modelModifiers?.capitalize) {
    value = value.charAt(0).toUpperCase() + value.slice(1)
  }
  emit('update:modelValue', value)
}
</script>

<template>
  <input :value="modelValue" @input="update(($event.target as HTMLInputElement).value)" />
</template>
```

`modelModifiers` prop — Vue avtomatik berib qo'yadi.

**Argument bilan multiple v-model'da:**

```vue
<MyComp v-model:title.capitalize="title" v-model:content.trim="content" />
```

```vue
<!-- MyComp.vue -->
<script setup lang="ts">
function capitalize(str: string): string {
  return str.charAt(0).toUpperCase() + str.slice(1)
}

const [title, titleModifiers] = defineModel<string>('title', {
  set(value) {
    return titleModifiers.capitalize ? capitalize(value) : value
  }
})

const [content, contentModifiers] = defineModel<string>('content', {
  set(value) {
    return contentModifiers.trim ? value.trim() : value
  }
})
</script>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Modifier compilation — parent:**

Template:
```vue
<MyComp v-model.capitalize="text" />
```

Compiled:

```javascript
return _createVNode(MyComp, {
  modelValue: _ctx.text,
  modelModifiers: { capitalize: true },
  "onUpdate:modelValue": $event => _ctx.text = $event
})
```

`modelModifiers` — Vue avtomatik qo'shadigan prop, modifier'lar object'i sifatida.

**Component runtime:**

```typescript
// Component setup ichida
const modifiers = props.modelModifiers || {}
console.log(modifiers.capitalize)  // true
```

**`defineModel({ set })` transform:**

`set` hook value emit qilinishidan oldin chaqiriladi. Return value emit qilinadi.

```typescript
// useModel customRef getter/set ichida options.get / options.set chaqiriladi
// (soddalashtirilgan — real source localValue + prevSetValue + prevEmittedValue saqlaydi)
function useModel<T>(props, name, options) {
  const i = getCurrentInstance()!
  let localValue: T = props[name]

  // prop o'zgarsa local buffer'ni sinxronlaydi
  watchSyncEffect(() => {
    const propValue = props[name]
    if (hasChanged(localValue, propValue)) {
      localValue = propValue
    }
  })

  return customRef((track, trigger) => ({
    get() {
      track()
      // local buffer'dan o'qiydi (props'dan to'g'ridan-to'g'ri emas), get transform bilan
      return options?.get ? options.get(localValue) : localValue
    },
    set(value: T) {
      const emittedValue = options?.set ? options.set(value) : value  // write transform
      if (hasChanged(value, localValue)) {
        localValue = value
        trigger()
      }
      i.emit(`update:${name}`, emittedValue)
    }
  }))
}
```

**`get` hook** (Vue 3.4+) — value parent'dan kelganda transform (reading):

```typescript
defineModel({
  get(value) {
    return value.toUpperCase()  // Display formatted
  },
  set(value) {
    return value.toLowerCase()  // Store lowercase
  }
})
```

Manba: [Vue.js Custom Modifiers](https://vuejs.org/guide/components/v-model.html#handling-v-model-modifiers)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Phone number — auto-format:**

```vue
<!-- PhoneInput.vue -->
<script setup lang="ts">
const [phone, modifiers] = defineModel<string>({
  set(value) {
    if (modifiers.format) {
      // Format: +XXX XX XXX XX XX
      const digits = value.replace(/\D/g, '')
      if (digits.length === 0) return ''
      const parts = [
        digits.slice(0, 3),     // country code
        digits.slice(3, 5),     // area
        digits.slice(5, 8),
        digits.slice(8, 10),
        digits.slice(10, 12)
      ].filter(Boolean)
      return '+' + parts.join(' ')
    }
    return value
  }
})
</script>

<template>
  <input v-model="phone" type="tel" placeholder="+998 90 123 45 67" />
</template>
```

Parent:

```vue
<PhoneInput v-model.format="phoneNumber" />
```

User "998901234567" yozsa, `phoneNumber = "+998 90 123 45 67"`.

**Capitalize first letter:**

```vue
<!-- NameInput.vue -->
<script setup lang="ts">
const [name, modifiers] = defineModel<string>({
  set(value) {
    return modifiers.capitalize
      ? value.split(' ').map(w => w.charAt(0).toUpperCase() + w.slice(1)).join(' ')
      : value
  }
})
</script>

<template>
  <input v-model="name" />
</template>
```

Parent:

```vue
<NameInput v-model.capitalize="fullName" />
<!-- "ali karimov" → "Ali Karimov" -->
```

</details>

---

## Edge Cases va Gotchas

### IME (Composition Event) handling

Chinese/Japanese/Korean input — multi-character composition mode. `v-model` avtomatik handle qiladi (compositionstart/end), value composition tugaganidan keyin yangilanadi:

```vue
<!-- v-model — IME safe -->
<input v-model="text" />

<!-- ❌ Manual @input — composition paytida ham trigger -->
<input :value="text" @input="text = $event.target.value" />
<!-- Bug: composition paytida ham state update -->
```

`v-model.lazy` — composition ahamiyatsiz (change event composition tugaganidan keyin).

### Number input — `null` vs `''` vs `0`

```vue
<script setup lang="ts">
import { ref } from 'vue'
const age = ref<number | null>(null)
</script>

<template>
  <!-- User input'ni clear qilsa: -->
  <input v-model.number="age" type="number" />
  <!-- age = '' (string) — looseToNumber('') NaN qaytaradi, original qoladi -->
</template>
```

`.number` modifier — empty string `''` qoladi (NaN check), `null` ga aylantirilmaydi. Manual handling kerak bo'lishi mumkin.

### `<textarea>` initial content ignored

```vue
<!-- ❌ Initial content text — ignored, v-model bo'sh string'ni set qiladi -->
<textarea v-model="text">Initial content here</textarea>

<!-- ✅ State'da initial value -->
<script setup lang="ts">
import { ref } from 'vue'

const text = ref('Initial content here')
</script>
<template>
  <textarea v-model="text"></textarea>
</template>
```

### Select default value — `<option disabled>`

```vue
<select v-model="country">
  <option disabled value="">Please select</option>  <!-- ← bu kerak -->
  <option value="uz">Uzbekistan</option>
  <option value="us">USA</option>
</select>
```

`v-model` initial value (`country = ''`) hech qaysi `<option>` value'siga mos kelmasa, `<select>` "unselected" holatda render bo'ladi. iOS Safari shu holatda birinchi tanlovda `change` event'ni trigger qilmaydi — natijada user birinchi option'ni tanlay olmaydi. Yechim: `value=""` bilan mos keladigan `<option disabled value="">` qo'shish — initial value unga mos keladi (`disabled` user tanlashidan saqlaydi). Vue docs iOS uchun shuni tavsiya qiladi.

### `v-model` qiymatining initial bo'lishi

```vue
<!-- ❌ Undefined initial — checkbox boshlanishda unchecked, lekin state aniq emas -->
<script setup lang="ts">
import { ref } from 'vue'

const flag = ref<boolean | undefined>(undefined)
</script>

<template>
  <input type="checkbox" v-model="flag" />
  <!-- undefined falsy → unchecked render; user bossa flag = true bo'ladi.
       TypeScript'da boolean | undefined union — keyingi `flag` ishlatishda undefined check majburiy -->
</template>
```

```vue
<!-- ✅ Initial boolean -->
<script setup lang="ts">
import { ref } from 'vue'

const flag = ref(false)
</script>

<template>
  <input type="checkbox" v-model="flag" />
</template>
```

### Custom value bilan checkbox — `true-value`/`false-value`

```vue
<!-- Boolean o'rniga custom value -->
<input type="checkbox" v-model="toggle" true-value="yes" false-value="no" />
<!-- toggle = "yes" (checked) yoki "no" (unchecked) -->
```

Dynamic binding:

```vue
<input type="checkbox" v-model="toggle" :true-value="customYes" :false-value="customNo" />
```

### `<select>` value type mismatch

```vue
<script setup lang="ts">
import { ref } from 'vue'

const id = ref<number>(0)
</script>

<template>
  <select v-model="id">
    <option value="1">One</option>  <!-- value="1" — STRING -->
    <option value="2">Two</option>
  </select>
  <!-- id = "1" (string!) — type mismatch -->
</template>
```

Yechim: `.number` modifier yoki `:value`:

```vue
<select v-model.number="id">
  <option value="1">One</option>
  <!-- id = 1 (number) -->
</select>

<!-- yoki dynamic value (number type saqlanadi) -->
<select v-model="id">
  <option :value="1">One</option>
  <option :value="2">Two</option>
</select>
```

---

## Common Mistakes

### Component'da `props.modelValue` ni mutate qilish

```vue
<!-- ❌ Component prop'ni o'zgartirish — TAQIQ -->
<script setup lang="ts">
const props = defineProps<{ modelValue: string }>()

function bad() {
  props.modelValue = 'changed'  // Vue warning: readonly
}
</script>
```

```vue
<!-- ✅ Emit orqali yangilash -->
<script setup lang="ts">
const props = defineProps<{ modelValue: string }>()
const emit = defineEmits<{ 'update:modelValue': [value: string] }>()

function correct(newValue: string) {
  emit('update:modelValue', newValue)
}
</script>
```

```vue
<!-- ✅✅ defineModel() bilan (3.4+) -->
<script setup lang="ts">
const model = defineModel<string>()
function update(value: string) {
  model.value = value  // Auto emit
}
</script>
```

### `v-model` o'rniga `:value` + `@input` qo'lda — typo xavfi

```vue
<!-- ❌ Verbose, typo xavfli -->
<input :value="text" @input="(e) => text = (e.target as HTMLInputElement).value" />

<!-- ✅ Vue magic -->
<input v-model="text" />
```

`v-model` IME, type cast, lazy mode'larni hammasini handle qiladi.

### Multiple checkbox — single boolean instead of array

```vue
<!-- ❌ Bir variable bir nechta checkbox uchun (boolean) -->
<script setup lang="ts">
import { ref } from 'vue'

const selected = ref(false)
</script>

<template>
  <input type="checkbox" value="a" v-model="selected" />
  <input type="checkbox" value="b" v-model="selected" />
  <!-- selected har bosishda toggle bo'ladi, qaysi value tanlanganini bilmaymiz -->
</template>
```

```vue
<!-- ✅ Array — har value alohida tanlanadi -->
<script setup lang="ts">
import { ref } from 'vue'

const selected = ref<string[]>([])
</script>

<template>
  <input type="checkbox" value="a" v-model="selected" />
  <input type="checkbox" value="b" v-model="selected" />
  <!-- selected = ['a'] yoki ['a', 'b'] -->
</template>
```

### Select option value type confusion

```vue
<!-- HTML attribute value har doim string -->
<select v-model="id">
  <option value="1">One</option>  <!-- "1" string -->
</select>
<!-- id = "1" (string), number kerak edi -->

<!-- ✅ Yechim 1: .number modifier -->
<select v-model.number="id">...</select>

<!-- ✅ Yechim 2: dynamic :value -->
<option :value="1">One</option>  <!-- 1 number -->
```

### `v-model` bilan `:value` birga

```vue
<!-- ❌ v-model va :value to'qnashadi -->
<input v-model="text" :value="otherText" />
<!-- :value override qiladi, lekin v-model two-way binding state'ni boshqaradi -->

<!-- ✅ Faqat birini ishlating -->
<input v-model="text" />
<!-- yoki -->
<input :value="text" @input="..." />
```

---

## Amaliy Mashqlar

### Mashq 1 [Junior]

Login form yarating: email, password, remember me checkbox. `v-model.trim` ishlatib email/password trim qiling. Submit'da console'ga form data yozing.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface LoginForm {
  email: string
  password: string
  rememberMe: boolean
}

const form = ref<LoginForm>({
  email: '',
  password: '',
  rememberMe: false
})

function handleSubmit() {
  console.log('Login data:', form.value)
}
</script>

<template>
  <form @submit.prevent="handleSubmit">
    <label>
      Email:
      <input v-model.trim="form.email" type="email" required />
    </label>
    <label>
      Password:
      <input v-model.trim="form.password" type="password" required />
    </label>
    <label>
      <input type="checkbox" v-model="form.rememberMe" />
      Remember me
    </label>
    <button type="submit">Login</button>
  </form>
</template>
```

`.trim` — user space yozsa ham, validation va backend'ga to'g'ri value yuboriladi.

</details>

### Mashq 2 [Middle]

Custom `TagInput` component yarating: user tag yozadi, Enter bilan tag qo'shadi (array'ga). Tag'ga click — o'chiradi. `v-model` orqali parent'ga array sync qiling (`defineModel()` ishlating).

<details>
<summary><strong>Yechim</strong></summary>

```vue
<!-- TagInput.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const tags = defineModel<string[]>({ default: () => [] })
const inputValue = ref('')

function addTag() {
  const trimmed = inputValue.value.trim()
  if (trimmed && !tags.value.includes(trimmed)) {
    tags.value = [...tags.value, trimmed]
    inputValue.value = ''
  }
}

function removeTag(index: number) {
  tags.value = tags.value.filter((_, i) => i !== index)
}
</script>

<template>
  <div class="tag-input">
    <span v-for="(tag, i) in tags" :key="tag" class="tag" @click="removeTag(i)">
      {{ tag }} ×
    </span>
    <input
      v-model="inputValue"
      @keydown.enter.prevent="addTag"
      placeholder="Add tag..."
    />
  </div>
</template>

<style scoped>
.tag-input {
  display: flex;
  flex-wrap: wrap;
  gap: 4px;
  padding: 4px;
  border: 1px solid #ccc;
  border-radius: 4px;
}
.tag {
  background: #3eaf7c;
  color: white;
  padding: 2px 8px;
  border-radius: 4px;
  cursor: pointer;
}
input { border: none; outline: none; flex: 1; min-width: 100px; }
</style>
```

Parent:

```vue
<script setup lang="ts">
import { ref } from 'vue'
import TagInput from './TagInput.vue'

const tags = ref<string[]>(['vue', 'typescript'])
</script>

<template>
  <TagInput v-model="tags" />
  <p>Tags: {{ tags.join(', ') }}</p>
</template>
```

</details>

### Mashq 3 [Middle+]

`DateRangePicker` komponent yarating: `from` va `to` date pickers, multiple `v-model` bilan (`v-model:from`, `v-model:to`). Validation: `to` `from`'dan keyin bo'lishi shart, aks holda error xabar.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<!-- DateRangePicker.vue -->
<script setup lang="ts">
import { computed } from 'vue'

const from = defineModel<string>('from', { default: '' })
const to = defineModel<string>('to', { default: '' })

const error = computed(() => {
  if (!from.value || !to.value) return null
  if (new Date(from.value) > new Date(to.value)) {
    return '"To" date must be after "From" date'
  }
  return null
})
</script>

<template>
  <div class="date-range">
    <label>
      From:
      <input v-model="from" type="date" />
    </label>
    <label>
      To:
      <input v-model="to" type="date" :min="from" />
    </label>
    <p v-if="error" class="error">{{ error }}</p>
  </div>
</template>

<style scoped>
.date-range { display: flex; gap: 12px; align-items: center; }
.error { color: red; }
</style>
```

Parent:

```vue
<script setup lang="ts">
import { ref } from 'vue'
import DateRangePicker from './DateRangePicker.vue'

const startDate = ref('')
const endDate = ref('')
</script>

<template>
  <DateRangePicker v-model:from="startDate" v-model:to="endDate" />
  <p>Range: {{ startDate }} → {{ endDate }}</p>
</template>
```

</details>

### Mashq 4 [Senior]

`CurrencyInput` komponent: `v-model` bilan number value, `defineModel({ set })` ishlatib auto-format. Custom `.format` modifier — `1234.5` → `"$1,234.50"` ko'rsatish, lekin underlying value number qoldirish.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<!-- CurrencyInput.vue -->
<script setup lang="ts">
import { computed, ref } from 'vue'

interface Props {
  currency?: string
  locale?: string
}

const props = withDefaults(defineProps<Props>(), {
  currency: 'USD',
  locale: 'en-US'
})

const [model, modifiers] = defineModel<number>({
  default: 0,
  set(value) {
    return typeof value === 'number' ? value : 0
  }
})

const inputValue = ref('')

// Display value computed — formatted
const displayValue = computed({
  get: () => {
    if (modifiers.format) {
      return new Intl.NumberFormat(props.locale, {
        style: 'currency',
        currency: props.currency,
        minimumFractionDigits: 2
      }).format(model.value)
    }
    return model.value.toFixed(2)
  },
  set: (val: string) => {
    // Format'dan number'ga parse
    const cleaned = val.replace(/[^\d.-]/g, '')
    const num = parseFloat(cleaned)
    if (!isNaN(num)) {
      model.value = num
    }
  }
})
</script>

<template>
  <input v-model="displayValue" type="text" />
</template>
```

Parent:

```vue
<script setup lang="ts">
import { ref } from 'vue'
import CurrencyInput from './CurrencyInput.vue'

const price = ref(1234.5)
</script>

<template>
  <CurrencyInput v-model.format="price" currency="USD" />
  <!-- Display: "$1,234.50", model: 1234.5 (number) -->
  <p>Stored value: {{ price }} ({{ typeof price }})</p>
</template>
```

`modifiers.format` true bo'lsa — formatted display, false bo'lsa — plain number.

</details>

### Mashq 5 [Senior]

`v-model` compilation'ni tushuntiring. Native input va component v-model bo'yicha compiled output yozing va asosiy farqlarni aniqlang.

```vue
<!-- A: Native input -->
<input v-model="text" />

<!-- B: Component -->
<MyInput v-model="text" />
```

<details>
<summary><strong>Yechim</strong></summary>

**A — Native input compilation:**

```vue
<input v-model="text" />
```

Compiled:

```javascript
import {
  vModelText as _vModelText,
  withDirectives as _withDirectives,
  createElementVNode as _createElementVNode
} from "vue"

export function render(_ctx, _cache) {
  return _withDirectives(
    _createElementVNode("input", {
      "onUpdate:modelValue": _cache[0] || (_cache[0] = $event => ((_ctx.text) = $event))
    }, null, 512 /* NEED_PATCH */),
    [[_vModelText, _ctx.text]]
  )
}
```

**Asosiy qismlar:**

1. **`vModelText` directive** — DOM input behavior (input event, IME, composition handling)
2. **`onUpdate:modelValue`** — synthetic event handler (update text variable)
3. **`withDirectives`** — directive'ni element'ga bog'lash
4. **`NEED_PATCH` flag** — runtime'ga directive lifecycle hook'larini chaqirish kerakligi haqida signal

**B — Component compilation:**

```vue
<MyInput v-model="text" />
```

Compiled:

```javascript
import { createVNode as _createVNode, resolveComponent as _resolveComponent } from "vue"

export function render(_ctx, _cache) {
  const _component_MyInput = _resolveComponent("MyInput")
  return _createVNode(_component_MyInput, {
    modelValue: _ctx.text,
    "onUpdate:modelValue": _cache[0] || (_cache[0] = $event => ((_ctx.text) = $event))
  })
}
```

**Asosiy qismlar:**

1. **`modelValue` prop** — component'ga reactive value uzatish
2. **`onUpdate:modelValue` handler** — component'dan emit event'ni listen qilish
3. **`resolveComponent`** — component nomini topish (global/local registration)

**Farqlar:**

| Aspect | Native input | Component |
|--------|--------------|-----------|
| **Mexanizm** | DOM directive (`vModelText`) | Props + emit pattern |
| **Compile output** | `withDirectives` + `vModelText` | `modelValue` prop + `onUpdate:modelValue` listener |
| **Runtime** | Directive lifecycle (mounted, updated, unmounted) | Component lifecycle, reactive props |
| **Customization** | DOM-specific (IME, composition) | Component logic (validation, transformation) |
| **TypeScript** | Implicit string | Explicit prop type |

**Native input — IME muhim:**

`vModelText` `compositionstart`/`compositionend` event'larni handle qiladi — Chinese, Japanese, Korean input'da composition mode'da DOM update'ni to'xtatadi.

**Component — flexibility:**

Component v-model — har qanday logic (validation, transformation, async update). `defineModel()` (Vue 3.4+) bilan ko'p boilerplate kamayadi.

**Cache pattern:**

Har ikkala variantda ham `_cache[0]` — handler bir marta yaratiladi, har render'da reuse. Bu `cacheHandlers: true` (default) optimization.

**Manba:** [Vue.js v-model compilation](https://vuejs.org/guide/components/v-model.html#under-the-hood), [`vModel` runtime source](https://github.com/vuejs/core/blob/main/packages/runtime-dom/src/directives/vModel.ts)

</details>

---

## Xulosa

`v-model` — Vue'ning two-way binding directive'i. Native input uchun (`:value` + `@input` shorthand), komponent uchun (`modelValue` prop + `update:modelValue` event). Vue compiler ikkalasini ham syntactic sugar sifatida transform qiladi.

Form element'lar — Vue avtomatik to'g'ri directive tanlaydi: `vModelText` (input/textarea), `vModelCheckbox`, `vModelRadio`, `vModelSelect`. Multiple checkbox'lar array bilan ishlaydi, select multi `multiple` attribute bilan array. IME (composition event) avtomatik handle qilinadi — multi-character input safe.

Modifier'lar: `.lazy` (change event, real-time emas), `.number` (string → number cast), `.trim` (whitespace olib tashlash). Chaining mumkin: `v-model.trim.number`. Custom modifier'lar (`v-model.capitalize`) — `defineModel({ set })` bilan transform.

Vue 3.4+ `defineModel()` macro — `v-model` boilerplate'ni keskin kamaytiradi. Eski pattern (`defineProps` + `defineEmits` + manual emit) 10+ qator bo'lsa, `defineModel()` 1 qator. Reactive `Ref` qaytaradi — local mutation parent state'ni emit qiladi.

Multiple `v-model` (Vue 3 native): `v-model:title`, `v-model:content` — bir component'da bir nechta two-way binding. Har biri alohida prop (`title`) + event (`update:title`). Real-world: date range picker, address form, multi-field input.

Common gotchas: `props.modelValue` mutate qilish TAQIQ (read-only); `<select>` value har doim string (`.number` modifier yoki dynamic `:value`); initial value `undefined` — checkbox/select issue; `<textarea>` initial content ignored (state'da set qilish kerak).

---

**Keyingi bo'lim:** [07-reactivity-fundamentals.md](07-reactivity-fundamentals.md) — Reactivity asoslari: `ref`, `reactive`, `toRef`/`toRefs`, `shallowRef`/`shallowReactive`.
