
# Frontend (Vue 3 + Pinia + Router + Vite) — Middle

### 🎯 Как рассуждать на собеседовании
- **Контекст:** SPA с картами/списками, требования по перформансу (LCP, TTI), стабильность навигации, предсказуемость состояния.
- **Подход:** “Сначала ясная архитектура и реактивная модель, затем оптимизация: мемоизация, виртуализация, lazy загрузка, кэширование API.”
- **Компромиссы:** где вынести логику в composables/store, где оставить в компоненте; когда SSR не нужен; когда debounce vs throttle.

---

## 1) Composition API на практике

**Когда использовать:** всегда, когда нужна переиспользуемая логика, типизация и читаемая структура.

**Ключевые инструменты:** `ref`, `reactive`, `computed`, `watch`, `watchEffect`, `toRef(s)`, `unref`, `onMounted`, `onUnmounted`, `defineProps`, `defineEmits`, `defineExpose`.

**Типичные паттерны:**
- **Локальное состояние** в компоненте + **бизнес-логика** в composables.
- **Коммуникация**: props → down, emits → up, provide/inject — для глубоких иерархий.

**Пример — поисковый блок с debounce и отменой:**
```vue
<script setup lang="ts">
import { ref, watch } from 'vue'

const query = ref('')
const isLoading = ref(false)
let controller: AbortController | null = null
let timer: number | null = null

async function search(q: string) {
  controller?.abort()
  controller = new AbortController()
  isLoading.value = true
  try {
    const res = await fetch(`/api/search?q=${encodeURIComponent(q)}`, { signal: controller.signal })
    // handle res.json()
  } finally {
    isLoading.value = false
  }
}

watch(query, (q) => {
  if (timer) clearTimeout(timer)
  timer = window.setTimeout(() => search(q), 300)
})
</script>
```

**Ошибка, которую часто ловят:** потеря реактивности при деструктуризации → используйте `toRefs`/`storeToRefs`.

---

## 2) Реактивность и производительность

- **`ref` vs `reactive`:** для примитивов и единичных значений — `ref`; для объектов — `reactive`. Не смешивайте лишний раз.
- **Глубокая vs поверхностная:** `shallowRef/shallowReactive` — когда не нужно отслеживать вложенности (например, большие неизменные объекты).
- **Оптимизация перерендеров:** `v-memo`, `v-once`, отсечка вычислений через `computed` (кеширование), `triggerRef` для ручного оповещения.
- **Большие списки:** **виртуализация** (Vue Virtual Scroller), `:key` стабилен, избегайте тяжелых вычислений внутри шаблона.

**Пример — мемоизация форматирования:**
```ts
const items = ref<Order[]>([])
const formatted = computed(() => items.value.map(x => ({
  ...x,
  sumFmt: Intl.NumberFormat('ru-RU').format(x.sum)
})))
```

---

## 3) Компоненты, props/emits/slots, expose

- **Props:** однонаправленный поток данных, задавайте типы и дефолты через `withDefaults`.
- **Emits:** четко именуйте события (`update:modelValue`, `select`, `close`).
- **Slots:** переиспользуемые UI-блоки, `scoped slots` для передачи данных родителю.
- **`defineExpose`** — аккуратно выдавайте наружу только то, что нужно (например, метод `focus()`).

**Пример — входной компонент с v-model:**
```vue
<script setup lang="ts">
const props = withDefaults(defineProps<{ modelValue: string; label?: string }>(), {
  label: 'Поиск'
})
const emit = defineEmits<{ (e: 'update:modelValue', v: string): void }>()
</script>

<template>
  <label>
    {{ props.label }}
    <input :value="props.modelValue" @input="e => emit('update:modelValue', (e.target as HTMLInputElement).value)" />
  </label>
</template>
```

---

## 4) Жизненный цикл и асинхронность

- **Где делать запросы:** `onMounted` / composables. Если нужна ранняя загрузка — в родительском компоненте и пробрасывать вниз.
- **Обработка ошибок:** показывайте пользователю статус (`isLoading`, `error`), логируйте через общую утилиту/интерцепторы.
- **keep-alive:** сохраняет состояние между переходами (подходит для списков, чтобы **сохранить позицию скролла**).

**Пример — безопасный запрос с отменой:**
```ts
import { onMounted, onBeforeUnmount } from 'vue'
let controller: AbortController | null = null

onMounted(async () => {
  controller = new AbortController()
  await fetch('/api/data', { signal: controller.signal })
})

onBeforeUnmount(() => controller?.abort())
```

---

## 5) Директивы, события, модификаторы

- `v-if` vs `v-show`: отрисовка vs скрытие (частое переключение → `v-show`).
- `v-for`: всегда задайте стабильный `:key`.
- Модификаторы событий: `.prevent`, `.stop`, `.once`, `.passive` — используйте по назначению, чтобы не блокировать поток.
- Кастомные директивы — для низкоуровневых DOM-задач (focus, click-outside).

---

## 6) Composables (useXxx) — организационный каркас

- Декомпозируйте бизнес-логику (поиск, фильтрация, карта, пагинация) в отдельные **composables**.
- **API:** аргументы — конфиг; возвращаем значения и функции. Не храните глобальное состояние внутри composable (если это не store).
- Тестируемость: чистые функции + возможность подменить зависимость (например, fetch).

**Пример — usePagination:**
```ts
import { ref, computed } from 'vue'
export function usePagination(pageSize = 20) {
  const page = ref(1)
  const total = ref(0)
  const skip = computed(() => (page.value - 1) * pageSize)
  function setTotal(t: number) { total.value = t }
  return { page, total, skip, pageSize, setTotal }
}
```

---

## 7) Pinia (state, getters, actions)

- **Когда store:** общего назначения и используется несколькими компонентами; состояние между страницами; кэширование результатов запросов; сложные фильтры.
- **Инструменты:** `defineStore`, `state/getters/actions`, `storeToRefs`, `$patch`, `$subscribe`.
- **Персист:** `pinia-plugin-persistedstate` для частичного сохранения (только нужные поля!).

**Пример — нормализованный store брендов:**
```ts
import { defineStore } from 'pinia'
export const useBrands = defineStore('brands', {
  state: () => ({ byId: {} as Record<number, Brand>, ids: [] as number[] }),
  getters: {
    list: (s) => s.ids.map(id => s.byId[id])
  },
  actions: {
    set(brands: Brand[]) {
      this.byId = Object.fromEntries(brands.map(b => [b.id, b]))
      this.ids = brands.map(b => b.id)
    }
  }
})
```

**Подводные камни:**
- Не мутируйте напрямую возвращённые из геттеров объекты; меняйте state через `actions`/`$patch`.
- Для тяжёлых вычислений — выносите в `computed` или кешируйте на сервере.

---

## 8) Vue Router — навигация, guards, lazy loading

- **Создание:** `createRouter`, `createWebHistory`.
- **Динамические маршруты:** `/brand/:id` → параметры через `useRoute()`.
- **Guards:** `beforeEach`, `beforeEnter` — проверка прав, загрузка данных, редиректы.
- **Lazy loading:** `component: () => import('...')` — уменьшает initial bundle.
- **Сохранение скролла:** `scrollBehavior` и/или keep-alive + восстановление позиции вручную.

**Пример — защита маршрута:**
```ts
router.beforeEach((to, from, next) => {
  if (to.meta.requiresAuth && !authStore.isLoggedIn) return next({ name: 'login' })
  next()
})
```

---

## 9) HTTP (fetch/Axios), интерцепторы, отмена, retry

- **Axios instance:** базовый URL, заголовки, интерцепторы для логирования и обработки ошибок.
- **Отмена запроса:** AbortController (fetch) или CancelToken (Axios).
- **Retry и таймаут:** через обертки/интерцепторы (или использовать `ky`/свои хелперы).

**Пример — Axios instance:**
```ts
import axios from 'axios'
export const api = axios.create({ baseURL: '/api', timeout: 8000 })
api.interceptors.response.use(
  r => r,
  e => {
    // централизованный хендлинг ошибок
    return Promise.reject(e)
  }
)
```

---

## 10) TypeScript в Vue 3 (ключевое для middle)

- Типизируйте **props**, **emits**, возвращаемые значения composables.
- Используйте **утилитные типы**: `Ref<T>`, `UnwrapRef<T>`, `ComponentPublicInstance`.
- В `defineProps`/`defineEmits` — **литералы**/генерики для точной типизации.
- Строгий режим TS в проекте (`"strict": true`) + алиасы в `tsconfig.json` (`paths: { "@/*": ["src/*"] }`).

**Пример — типизация emits:**
```ts
const emit = defineEmits<{
  (e: 'select', id: number): void
  (e: 'close'): void
}>()
```

---

## 11) Формы и валидация

- Нативная валидация + кастомные ошибки.
- **VeeValidate** / **Vuelidate** — схемы валидации, ответы сервера → маппим на поля.
- Масштабирование: компонент `FormField` со слотами для сообщений об ошибке.

**Пример — VeeValidate:**
```vue
<Form @submit="onSubmit">
  <Field name="email" rules="required|email" v-slot="{ field, errorMessage }">
    <input v-bind="field" />
    <span v-if="errorMessage">{{ errorMessage }}</span>
  </Field>
</Form>
```

---

## 12) CSS/SCSS, архитектура стилей

- **scoped** стили в SFC для локализации.
- Глобальные переменные (например, `--navBarHeight`) — для лейаута и темизации.
- Архитектуры: **BEM**, **Utility-first** (Tailwind) или аккуратный SCSS-модули.
- Избегайте утечек стилей: нейминг по блокам, без глобальных каскадных хаков.

**Пример — class binding:**
```vue
<div :class="['badge', { 'badge--active': isActive }]">{{ text }}</div>
```

---

## 13) Тестирование: Vitest + @vue/test-utils

- **Unit**: логика composables, методы компонентов, отрисовка элементарных UI.
- **Integration**: взаимодействие компонентов, роутер, store.
- Моки API, таймеров, localStorage.
- Сквозные e2e: **Cypress/Playwright** (минимум — smoke сценарии).

**Пример — тест компонента:**
```ts
import { mount } from '@vue/test-utils'
import MyButton from './MyButton.vue'

it('emits click', async () => {
  const w = mount(MyButton)
  await w.get('button').trigger('click')
  expect(w.emitted('click')).toBeTruthy()
})
```

---

## 14) Vite: конфиг, алиасы, env

- **vite.config.ts:** алиасы, плагины, оптимизация зависимостей.
- **.env** файлы: `VITE_*` переменные доступны в клиенте (`import.meta.env.VITE_API_BASE`).
- **Code splitting:** динамические импорты, `build.rollupOptions` для тонкой настройки.

**Пример — алиас “@”:**
```ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import path from 'path'

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: { '@': path.resolve(__dirname, 'src') }
  }
})
```

---

## 15) Производительность и UX

- **Lazy** компоненты/роуты, `prefetch`/`preload` критичных ресурсов.
- **Debounce/throttle** для событий скролла/ввода.
- **Virtual Scrolling** для длинных списков (карты, точки).
- Избегайте ненужной реактивности: большие статичные данные как `shallowRef/markRaw`.
- **Web Vitals:** следите за LCP/CLS/FID; избегайте layout shift (фиксируйте размеры изображений/карт).

**Пример — throttle для скролла:**
```ts
let busy = false
window.addEventListener('scroll', () => {
  if (busy) return
  busy = true
  // do work
  setTimeout(() => (busy = false), 100)
})
```

---

## 16) Доступность (a11y)

- Семантическая разметка: правильные теги, `<button>` для действий (а не `div`).
- Фокус-менеджмент в модалках: вернуть фокус после закрытия.
- ARIA-атрибуты: `role`, `aria-label` для иконок и нестандартных контролов.
- Тестирование: axe-core/Lighthouse.

---

## 17) Безопасность фронтенда

- **XSS:** не используйте `v-html` с непроверенным вводом; санитайз на сервере/клиенте.
- **CSRF:** для cookie-сессий используйте защищенные флаги (`HttpOnly`, `SameSite`), токены запроса.
- **CSP:** настройка Content-Security-Policy (block inline-скрипты, allowlist доменов).
- **Хранение токенов:** предпочтительно **httpOnly cookie**; если `localStorage` — то с коротким TTL и строгой политикой.

---

## 18) Частые вопросы middle и краткие ответы

- **IEnumerable vs Observable в Vue?** (Трюк): реактивность Vue — это Proxy-трекинг зависимостей; массивы отслеживаются корректно, но мутации должны идти через реактивные методы (`push`, `splice`) или замену ссылки.
- **Почему `v-if` медленнее `v-show`?** `v-if` пересоздаёт DOM; `v-show` только меняет `display`. При частых переключениях — `v-show`.
- **Почему теряется позиция скролла при возврате на страницу?** Без keep-alive/scrollBehavior состояние компонента пересоздаётся; храните позицию в store/route meta и восстанавливайте.
- **Как уменьшить лаг при отрисовке карты/точек?** Кластеризация, виртуализация, разделение на слои, отложенная загрузка метаданных.

---

## 19) Самопроверка (перед интервью)

- Могу объяснить отличие `ref/reactive`, `computed/watch`, shallow реактивности.
- Знаю, как организовать composables и Pinia store для кэширования и фильтров.
- Умею настраивать Router с guards и lazy loading.
- Понимаю Axios instance, интерцепторы, отмену, обработку ошибок.
- Умею улучшать перформанс: виртуализация, debounce/throttle, lazy, мемоизация.
- Помню про a11y и безопасность (XSS, CSP, хранение токенов).
- Могу написать тест на компонент/компоновку состояния.
