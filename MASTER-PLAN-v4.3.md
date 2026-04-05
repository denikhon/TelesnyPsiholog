# 🗺️ MASTER-PLAN — Scene Engine v4.3
### Сайт: Анастасия Швейкина — телесно-ориентированный психолог
> ⚠️ Этот файл — эталонный документ. Не перезаписывать implementation plan'ами.
> После каждого прохода разработки сверяться с этим планом.

---

## Структура сцен (финальная, согласована)
```js
SCENES = [
  { id: 1, textBlocks: 0, isIntro: true },           // 1.webp — статичная интро
  { id: 2, textBlocks: 2 },                           // 2.webp — запросы клиентов
  { id: 3, textBlocks: 1 },                           // 3.webp — подход
  { id: 4, textBlocks: 2 },                           // 4.webp — как работают сессии
  { id: 5, textBlocks: 1, hasCarousel: true },        // 5.webp — цены + карусель
  { id: 6, textBlocks: 1, hasButtons: true },         // p2.webp — контакты
]
// После сцен 1–6 идёт отдельная секция <section id="faq"> — вне сцен,
// вне scroll-trigger зоны, без split-механики.
```

> ⚠️ ВАЖНО: Сцена 5 имеет `textBlocks: 1` (цены) + `hasCarousel: true` (карусель отзывов).
> Карусель (`scene__carousel-slot`) рендерится как **последний дочерний элемент `.scene__text-layer`** (Вариант Б).
> Это гарантирует, что `scrollHeight` текстового слоя автоматически включает высоту карусели —
> формула `getSceneHeight` остаётся универсальной без ветвления по `sceneConfig`.
> Карусель не входит в GSAP-timeline (селектор `.scene__text-block` её не захватывает).

> ⚠️ ВАЖНО: FAQ (`hasFaq`) **удалён из конфига сцены 6**.
> FAQ рендерится как `<section id="faq">` после `<main id="scenes-container">` — отдельно.
> Это исключает FAQ из расчёта `getSceneHeight` сцены 6 и из зоны ScrollTrigger.
> `ScrollTrigger.refresh()` при открытии FAQ не нужен — убран из `initFaq()`.

---

## Файловая структура проекта
```
/
├── index.html
├── content.json          ← единственный источник контента
├── css/
│   ├── reset.css
│   ├── tokens.css        ← CSS-переменные (цвета, шрифты, отступы)
│   ├── scene.css         ← стили сцен
│   ├── carousel.css      ← стили карусели
│   ├── faq.css           ← стили FAQ (секция вне сцен)
│   └── buttons.css       ← стили кнопок
├── js/
│   ├── main.js           ← точка входа, lifecycle
│   ├── config.js         ← SCENES, глобальные константы
│   ├── content.js        ← fetch + validateContent()
│   ├── render.js         ← renderScenes() + renderFaq() + buildSceneElementsMap()
│   ├── engine.js         ← initScenes(), initScene(), recalcSceneHeights()
│   ├── carousel.js       ← initCarousel(), touch-логика
│   ├── faq.js            ← initFaq()
│   └── images.js         ← initImageLoading() (IntersectionObserver)
└── public/
    ├── 1.webp–5.webp     ← фоновые изображения сцен 1–5
    ├── p2.webp           ← фоновое изображение сцены 6
    └── reviews/
        └── 1.webp–6.webp ← слайды карусели
```

---

## Фазы разработки

---

### ФАЗА 0 — Scaffolding (каркас)
**Цель:** пустой HTML-каркас + CSS-система + подключение библиотек

#### Задачи:
- [ ] `index.html` — базовая разметка, подключение GSAP + ScrollTrigger через CDN
- [ ] `css/reset.css` — современный CSS reset
- [ ] `css/tokens.css` — CSS-переменные: шрифт (Google Fonts), палитра, отступы
- [ ] `css/scene.css` — базовые стили `.scene`, `.scene__mask`, `.scene__title-layer`, `.scene__text-layer`, `.scene__text-block`
- [ ] `js/config.js` — все глобальные константы и массив `SCENES`:

```js
const SCRUB                = 0.8;
const SPLIT_END            = 0.75;  // прогресс сцены когда clip-path полностью открыт
const TEXT_START           = 0.65;  // прогресс когда первый текстблок начинает движение
                                    // overlap = SPLIT_END - TEXT_START = 0.10 (намеренно)
const MAX_TEXT_LINES       = 10;
const RESIZE_DEBOUNCE      = 250;
const IMAGE_PRELOAD_MARGIN = '200%';
// BASE_VH не константа в config — вычисляется в main() как window.innerHeight
// и передаётся в initScene() параметром, чтобы зафиксировать пиксели при инициализации
```

#### Проверка фазы 0:
- Страница открывается в браузере без ошибок в консоли
- CSS-переменные доступны (можно проверить через DevTools)

---

### ФАЗА 1 — Контент и рендер
**Цель:** загрузить content.json, отрендерить DOM-структуру всех сцен и FAQ

#### Задачи:
- [ ] `js/content.js` — `fetchContent()`, `validateContent(content, SCENES)`

  **`fetchContent()` — обработка ошибок обязательна:**
  ```js
  async function fetchContent() {
    let response;
    try {
      response = await fetch('/content.json');
    } catch (e) {
      showFatalError('Не удалось загрузить контент. Проверьте соединение.');
      throw e;
    }
    if (!response.ok) {
      showFatalError(`Ошибка загрузки контента: ${response.status}`);
      throw new Error(`HTTP ${response.status}`);
    }
    let content;
    try {
      content = await response.json();
    } catch (e) {
      showFatalError('Контент повреждён (невалидный JSON).');
      throw e;
    }
    return content;
  }

  function showFatalError(message) {
    // Единственное место где допустим inline style и прямая запись в body.
    // Это намеренный аварийный режим до загрузки CSS.
    document.body.innerHTML =
      `<div style="padding:2rem;font-family:sans-serif">${message}</div>`;
  }
  ```

  **Что валидирует `validateContent`:**
  - Для каждой не-intro сцены: наличие `image`, `title`, совпадение количества `textBlocks`
  - Длина заголовка: если заголовок > 3 строк — `console.warn()` (Риск 7)
  - Количество строк в каждом textBlock ≤ `MAX_TEXT_LINES`

- [ ] `js/render.js` — `renderScenes(content, SCENES)` + `renderFaq(content.faq)`:

  **DOM-структура одной сцены (сцены 2–6):**
  ```html
  <div class="scene" data-scene="N">
    <div class="scene__mask">
      <img data-src="N.webp" alt="">
      <!-- НЕТ отдельного .scene__image-layer — img прямо в .scene__mask -->
    </div>
    <div class="scene__title-layer">
      <h2 class="scene__title">...</h2>
    </div>
    <div class="scene__text-layer">
      <div class="scene__text-block">...</div>
      <!-- для сцены 5: последним элементом идёт .scene__carousel-slot -->
    </div>
  </div>
  ```

  **FAQ рендерится отдельно — после `#scenes-container`:**
  ```js
  function renderFaq(faqData) {
    const section = document.getElementById('faq');
    // рендер аккордеона в #faq
  }
  ```
  ```html
  <!-- index.html -->
  <main id="scenes-container">
    <!-- сцены 1–6 -->
  </main>
  <section id="faq">
    <!-- renderFaq() монтирует сюда -->
  </section>
  ```

  - Сцена 1 (isIntro) — специальная разметка (см. §8 — Интро-сцена)
  - Сцена 5 — добавить `.scene__carousel-slot` как **последний дочерний элемент `.scene__text-layer`**
  - Сцена 6 — добавить кнопки контактов (`.contact-btn` × N) из `content.buttons`
  - `buildSceneElementsMap()` — кэш DOM-элементов в `Map`
  - Установка `--scene-index` через `sceneEl.style.setProperty('--scene-index', i + 1)`

- [ ] `js/images.js` — `initImageLoading()`:
  - Сцена 1: `loading="eager" fetchpriority="high"` — img получает `src` сразу
  - Сцены 2–6: `data-src` на img внутри `.scene__mask`, `IntersectionObserver` с `rootMargin: IMAGE_PRELOAD_MARGIN` (вертикальный)
  - `img.decoding = "async"` при загрузке

#### Проверка фазы 1:
- В браузере видна разметка всех 6 сцен + секция `#faq` после них
- `validateContent()` бросает ошибку при нарушении структуры
- Консоль чистая (нет 404 на изображения до IntersectionObserver)
- При недоступном `/content.json` — страница показывает читаемое сообщение об ошибке

---

### ФАЗА 2 — CSS сцен (визуальная основа)
**Цель:** сцены физически перекрывают друг друга, sticky-маска залипает на viewport, картинки отображаются без искажений

#### Ключевая механика (обязательно понять перед кодингом):
1. Каждая следующая сцена имеет больший `z-index` и физически наезжает **поверх** предыдущей через `margin-top: -100dvh`
2. Но мы её не видим — она скрыта `clip-path: inset(100% 0 0 0)`
3. `.scene__mask` — `position: sticky` — залипает на viewport пока пользователь скроллит длинный контейнер `.scene`
4. GSAP раскрывает `clip-path` снизу вверх → эффект шторки

#### Задачи:
- [ ] `css/scene.css` — все слои сцены:

```css
/* ─── Контейнер сцены ───────────────────────────────────────────────────────
   Запрещено: transform, overflow: hidden, filter, clip-path — создают
   stacking context и убивают position: sticky дочерних элементов.
   contain: layout style — убран (Риск 1: ломает sticky/scrollHeight в Safari) */
.scene {
  position: relative;
  /* height выставляется явно через JS после recalcSceneHeights() */
  z-index: calc(var(--scene-index) * 100);
}

/* ─── Перекрытие сцен ───────────────────────────────────────────────────────
   Каждая сцена (кроме первой) наезжает на предыдущую ровно на один viewport.
   Селектор .scene ~ .scene не затрагивает первую сцену — она первая в потоке
   и не имеет предшествующего .scene. Это намеренно: интро стоит без перекрытия.
   БЕЗ этого margin z-index бессмысленен — сцены стоят в потоке одна за другой. */
.scene ~ .scene {
  margin-top: -100dvh;
}

/* ─── Маска (sticky-контейнер для картинки) ─────────────────────────────────
   [FIX v4.3] position: sticky — маска залипает на viewport.
   position: absolute (старый план) уезжала вместе с потоком .scene —
   шторка не работала, между картинками образовывались дыры.

   overflow: hidden — обязателен: без него img (position: absolute, inset: 0)
   вылезает за пределы 100dvh sticky-контейнера при высоте .scene 3000px+.

   ⚠️ Safari: sticky ломается если любой ancestor имеет overflow: hidden/auto.
   Убедиться что на .scene, body, html — нет overflow: hidden. (Риск 11) */
.scene__mask {
  position: sticky;
  top: 0;
  height: 100dvh;
  overflow: hidden;
  clip-path: inset(100% 0 0 0);
  z-index: 0;
}

/* ─── Картинка ──────────────────────────────────────────────────────────────
   [FIX v4.3] object-fit: cover + inset: 0 вместо transform: translateY(-50%).
   translateY(-50%) растягивал вертикальные изображения на широких экранах —
   width: 100% задавал гигантскую ширину, height масштабировался пропорционально.
   img живёт прямо в .scene__mask — отдельный .scene__image-layer не нужен. */
.scene__mask img {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;
  object-position: center;
}

/* ─── Sticky заголовок ──────────────────────────────────────────────────────
   position: sticky даёт нужный эффект без дополнительной GSAP-анимации:
   заголовок "въезжает" синхронно с раскрытием clip-path (оба привязаны
   к скроллу одного контейнера .scene) и залипает у верха по достижении top.
   Отдельная GSAP-анимация заголовка — не нужна и не должна добавляться.
   Нет overflow: hidden — sticky работает только вне clip-path контекста. */
.scene__title-layer {
  position: sticky;
  top: 32px;
  z-index: 2;
  max-width: 600px;
  margin: 0 auto;
}

/* ─── Текстовый слой ────────────────────────────────────────────────────────
   scrollHeight этого элемента используется в getSceneHeight().
   Для сцены 5: .scene__carousel-slot внутри этого слоя —
   scrollHeight автоматически включает высоту карусели. */
.scene__text-layer {
  position: relative;
  z-index: 2;
  padding-top: calc(var(--title-height, 60px) + 64px);
}
```

- [ ] Сцена интро (`.scene--intro`) — см. подробное описание в §8

- [ ] z-index проверить: сцена 6 (z-index: 600) перекрывает сцену 5 (z-index: 500) и т.д.

#### Проверка фазы 2 — тест без JS (обязательно пройти перед Фазой 3):
```
1. Отключить все скрипты Фазы 3 (GSAP, ScrollTrigger)
2. Закомментировать clip-path: inset(100% 0 0 0) на .scene__mask

Ожидаемое поведение при скролле:
✓ Картинки залипают на viewport (sticky работает) —
  каждая держится на экране пока скроллится её контейнер .scene
✓ Сцены наползают друг на друга без разрывов (margin-top: -100dvh)
✓ Порядок перекрытия верный: сцена 2 поверх сцены 1 (z-index)
✓ Пропорции картинок корректны, нет растяжки (object-fit: cover)
✓ При скролле до конца сцены 1 — картинка сцены 2 наползает снизу
  и тоже залипает. Никаких дыр и разрывов между картинками.

❌ Если видны дыры между картинками →
   sticky не работает или margin-top: -100dvh отсутствует
❌ Если картинки растянуты/увеличены →
   убедиться что нет translateY, есть overflow: hidden на .scene__mask,
   есть object-fit: cover на img
❌ Sticky не работает в Safari →
   проверить отсутствие overflow: hidden на .scene, body, html (Риск 11)

Только после чистого прохождения теста — переходить к Фазе 3.
```

---

### ФАЗА 3 — Высоты сцен + Scene Engine
**Цель:** реализовать вычисление высот, явное выставление `style.height`, инициализацию GSAP

#### Задачи:
- [ ] `js/engine.js` — `getSceneHeight(sceneEl, sceneConfig)`:
```
BASE + SPLIT + textHeight
BASE      = window.innerHeight
SPLIT     = window.innerHeight
textHeight = sceneEl.querySelector('.scene__text-layer')?.scrollHeight || 0
```
  > ⚠️ `getSceneHeight` вызывается только ПОСЛЕ `document.fonts.load()` (шаг 6 lifecycle)
  >
  > ✔ Для сцены 5: `.scene__carousel-slot` внутри `.scene__text-layer` →
  > `scrollHeight` автоматически включает высоту карусели — формула универсальна.
  >
  > ✔ Для сцены 6: FAQ вынесен в отдельную `<section id="faq">` вне `.scene` →
  > `scrollHeight` сцены 6 не зависит от состояния аккордеона.
  >
  > ⚠️ Геометрия страницы с учётом перекрытий:
  > `totalPageHeight = sum(sceneHeights) - (SCENES.length - 1) * BASE_VH`
  > Это ожидаемое поведение (каждая сцена кроме первой наезжает на -100dvh).
  > При отладке общей высоты страницы учитывать этот вычет.

- [ ] `recalcSceneHeights()` — пересчёт массива `sceneHeights[]`

- [ ] Явное выставление высоты DOM-элементов:
```js
sceneEl.style.height = sceneHeights[i] + 'px';
```

- [ ] `initScene(sceneEl, sceneConfig, prevSceneEl, sceneHeight)`:
  - Все пороговые значения берутся из `config.js` — не магические числа:
  ```js
  // SPLIT_END, TEXT_START импортируются из config.js
  const splitRatio   = SPLIT_END;                        // 0.75
  const overlapRatio = SPLIT_END - TEXT_START;           // 0.10
  const textStartPos = TEXT_START;                       // 0.65
  ```
  - `BASE_VH` — фиксированное число пикселей, не CSS-строка:
  ```js
  // [FIX v4.3] BASE_VH фиксируется при инициализации как window.innerHeight.
  // Строка '100vh' вызывает jank на мобильных: браузер пересчитывает значение
  // при скрытии/показе адресной строки.
  const BASE_VH = window.innerHeight;
  ```
  - Фаза 1 timeline: clip-path маски (`0 → splitRatio`, `ease: "none"`)
  - Фаза 2 timeline: последовательный вывод textBlocks

  **Механика nth текстблока (точная триггерная логика):**
  ```
  Текстблок N начинает движение снизу в момент когда верхний край
  текстблока (N-1) пересекает top: 0 страницы.
  Блоки идут встык — второй стартует ровно когда первый "ушёл" за верх viewport.

  В терминах timeline-прогресса:
  blockStart[0] = textStartPos                              // = TEXT_START = 0.65
  blockStart[N] = blockStart[N-1] + (blockHeight[N-1] / sceneHeight)
  // blockHeight — реальная offsetHeight элемента .scene__text-block в пикселях,
  // переведённая в долю от общей высоты сцены (sceneHeight).
  // Вызывать после document.fonts.load() и после style.height установлен.
  ```

  - **Буфер**: после последнего textBlock остаётся ~20–40% прогресса без анимации — это ожидаемое поведение (зона чтения перед переходом к следующей сцене)
  - Фаза 3 timeline: кнопки сцены 6 (`hasButtons`) — появляются как статичные горизонтальные прямоугольники, никакой track/scaleX анимации не нужно. Кнопки просто присутствуют в DOM и становятся видимы вместе с текстблоком сцены 6 через общий поток сцены. Отдельная GSAP-анимация для кнопок — не добавлять.
  - Отдельный ScrollTrigger для `will-change`:
    - Привязан к `sceneEl`, зона `start: "top 120%"` / `end: "bottom -20%"`
    - `onEnter` → `will-change: clip-path` на `.scene__mask`
    - `onLeave` + `onLeaveBack` → `will-change: auto` (снятие обязательно — иначе утечка GPU)

- [ ] `initScenes()`:
  - `gsap.set(".scene:not([data-scene='1']) .scene__mask", { clipPath: "inset(100% 0 0 0)" })`
  - Итерация SCENES, вызов `initScene()` для каждой не-intro сцены

#### Проверка фазы 3:
- Скролл вниз раскрывает сцены через маску (снизу вверх)
- Маска раскрывается строго в пределах одного экрана
- Заголовки sticky — прилипают к верху при прокрутке внутри сцены
- Первый текстблок начинает движение снизу при прогрессе 0.65 сцены
- Второй текстблок стартует снизу ровно когда верхний край первого уходит за top: 0
- Скролл вверх точно инвертирует анимацию
- `will-change` в DevTools → Layers: активируется только для текущей/следующей сцены и снимается при выходе из зоны

---

### ФАЗА 4 — Карусель (Сцена 5)
**Цель:** горизонтальный свайп отзывов, не конфликтующий с вертикальным скроллом

#### Задачи:
- [ ] `css/carousel.css` — стили карусели (`.carousel`, `.carousel__track`, `.carousel__slide`)
- [ ] `js/carousel.js` — `initCarousel(slides)`:
  - DOM-клонирование для loop (первые 2 → конец, последние 2 → начало)
  - Guard: если `slides.length < 3` → `disableLoop()` (у нас 6 слайдов → loop активен)
  - Клоны создаются через `cloneNode(true)` — `data-src` копируется автоматически, `src` у клонов нет

  **Lazy-loading слайдов — отдельный Observer (не images.js):**
  ```
  Два независимых IntersectionObserver:
  1. images.js — вертикальный, rootMargin: '200%' — для фоновых .scene__mask img
  2. carousel.js — горизонтальный, rootMargin: '0px 200px 0px 200px' — для слайдов

  Логика carousel Observer:
  - observe() вызывается для всех слайдов включая клоны
  - При repositioning (loop) — повторная observe() для переставленных клонов
  - При пересечении: img.src = img.dataset.src; observer.unobserve(img)
  ```

  - Repositioning только когда карусель в покое (не в момент анимации)

- [ ] Touch-логика карусели:
```css
.carousel { touch-action: pan-x; }
```
```js
// touchstart: passive: true
// touchmove: passive: false, с e.cancelable проверкой перед preventDefault
// isHorizontalSwipe определяется по dx > dy && dx > 8
```

#### Проверка фазы 4:
- Горизонтальный свайп работает внутри сцены 5
- Вертикальный скролл не блокируется при свайпе карусели
- Loop бесконечный (repositioning без прыжков)
- Клоны загружают изображения через горизонтальный Observer
- В консоли нет `Unable to preventDefault inside passive event listener`

---

### ФАЗА 5 — FAQ (секция после сцен)
**Цель:** аккордеон для вопросов/ответов, изолированный от scroll-trigger зоны сцен

#### Архитектура:
FAQ — отдельная `<section id="faq">` после `<main id="scenes-container">`.
Находится вне зоны ScrollTrigger. Изменение высоты FAQ не влияет на триггеры сцен.
`ScrollTrigger.refresh()` при открытии аккордеона — **не нужен**.

#### Задачи:
- [ ] `css/faq.css` — стили FAQ (`.faq`, `.faq__item`, `.faq__question`, `.faq__content`)
- [ ] `js/faq.js` — `initFaq()`:
  - Аккордеон с возможностью открыть несколько пунктов одновременно
  - Работает с `#faq` секцией — не с DOM сцен
  - `ScrollTrigger.refresh()` при `transitionend` — **убран** (не нужен при изолированном FAQ)
- [ ] `js/render.js` — `renderFaq(content.faq)` монтирует разметку в `#faq`

#### Проверка фазы 5:
- FAQ раскрывается/закрывается плавно
- Открытие нескольких пунктов — работает
- При открытии/закрытии FAQ сцены выше не "прыгают"
- FAQ не является частью ни одной `.scene`

---

### ФАЗА 6 — Resize + Reduced Motion + ScrollTrigger.refresh()
**Цель:** стабильность при изменении размера окна и поддержка accessibility

#### Задачи:
- [ ] `initResizeHandler()`:
  - Guard: `Math.abs(window.innerWidth - lastWidth) < 2` → skip (защита от субпиксельного масштабирования Android)
  - Debounce: `RESIZE_DEBOUNCE` (250ms)
  - Порядок: `ScrollTrigger.kill()` → `recalcSceneHeights()` → `scene.style.height` → `initScenes()`
  - После `ScrollTrigger.kill()` и до `initScenes()` маски находятся в неопределённом состоянии — `gsap.set(".scene__mask", { clipPath: "inset(100% 0 0 0)" })` вызывается внутри `initScenes()` и восстанавливает состояние

- [ ] `prefers-reduced-motion`:
```js
if (prefersReducedMotion) {
  // Показать все элементы статично
  gsap.set(".scene__mask", { clipPath: "inset(0% 0 0 0)" });
  gsap.set(".scene__text-block", { y: 0 });
  // Кнопки сцены 6 — статичные прямоугольники, GSAP не управляет ими,
  // дополнительный gsap.set для кнопок не нужен
} else {
  initScenes();
}
```

- [ ] `ScrollTrigger.refresh()` в конце `main()` (после всех инициализаций)

#### Проверка фазы 6:
- Поворот экрана → сцены пересчитываются корректно, нет прыжков
- `prefers-reduced-motion: reduce` → все сцены и кнопки видны статично
- `ScrollTrigger.refresh()` вызывается один раз в конце `main()`

---

### ФАЗА 7 — Desktop версия
**Цель:** отдельный интерфейс для desktop через `matchMedia`

> ⚠️ Desktop-версия разрабатывается отдельно после завершения и тестирования мобильной.
> Фаза 7 — плейсхолдер. Детальный план будет написан отдельным документом.

#### Задачи (контуры):
- [ ] Переключение через `window.matchMedia('(pointer: fine) and (min-width: 1024px)')`
- [ ] Desktop: двухколоночный layout (Image | Text), обычный scroll без GSAP
- [ ] Отдельные CSS-файлы для desktop (`css/desktop.css`)
- [ ] **Правило изоляции:** ни один desktop-компонент не импортирует mobile-логику и наоборот
- [ ] Общий `content.json`, раздельные рендеры

#### Проверка фазы 7:
- На desktop (pointer: fine, ширина ≥ 1024px) — показывается desktop-версия
- На мобильном — показывается mobile Scene Engine
- Нет импортов между mobile/desktop JS

---

---

## §8 — Интро-сцена (Сцена 1): архитектура и изоляция

### Принцип изоляции
Сцена 1 — полностью статичная. Она **не участвует** ни в одной из механик сцен 2–6:
- Нет clip-path анимации (нет `.scene__mask` с sticky)
- Нет GSAP ScrollTrigger на этой сцене
- Нет sticky-заголовка по механике сцен 2–6
- Нет текстблоков по механике engine.js
- `isIntro: true` в SCENES — engine.js пропускает её при `initScenes()`

Визуально заголовок и текстблок интро **выглядят идентично** заголовкам и текстблокам остальных сцен (те же классы, те же токены), но являются статичными позиционированными элементами внутри одного экрана.

### DOM-структура сцены 1
```html
<div class="scene scene--intro" data-scene="1">
  <div class="scene__intro-image">
    <img src="1.webp" alt="" loading="eager" fetchpriority="high">
  </div>
  <div class="scene__intro-title">
    <h1 class="scene__title">...</h1>
    <!-- Визуально идентичен .scene__title сцен 2–6, но position: static в потоке -->
  </div>
  <div class="scene__intro-text">
    <div class="scene__text-block">...</div>
    <!-- Визуально идентичен .scene__text-block сцен 2–6, но статичный -->
  </div>
</div>
```

### CSS сцены 1
```css
/* Интро занимает ровно один экран */
.scene--intro {
  position: relative;
  height: 100dvh;
  display: flex;
  flex-direction: column;
  /* НЕ получает margin-top: -100dvh — стоит первой в потоке без перекрытия */
  /* z-index: calc(var(--scene-index) * 100) наследуется = z-index: 100 */
}

/* Картинка прикреплена к верхнему краю, занимает всё пространство кроме текстблока */
.scene__intro-image {
  position: absolute;
  inset: 0;
  z-index: 0;
}
.scene__intro-image img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  object-position: center;
}

/* Заголовок — поверх картинки, в верхней зоне */
.scene__intro-title {
  position: relative;
  z-index: 1;
  padding: 32px 24px 0;
  /* Визуально совпадает с .scene__title-layer по шрифту и отступам */
}

/* Текстблок — прижат к нижнему краю экрана, занимает оставшееся пространство */
.scene__intro-text {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  z-index: 1;
  /* Визуально совпадает с .scene__text-block по стилям плашки */
}
```

### Что не делать для сцены 1
- Не добавлять `position: sticky` на любой дочерний элемент — не нужно
- Не добавлять `.scene__mask` — нет clip-path механики
- Не включать в `initScenes()` — engine.js проверяет `sceneConfig.isIntro` и пропускает
- Не добавлять `style.height` через `recalcSceneHeights()` — высота фиксирована CSS (`100dvh`)
- Не добавлять `data-src` на img — картинка грузится eager немедленно

### Проверка сцены 1
- Занимает ровно 100dvh без скролла
- Картинка не искажена (object-fit: cover)
- Заголовок читается поверх картинки
- Текстблок прижат к нижнему краю экрана
- При скролле вниз сцена 2 наползает поверх сцены 1 (за счёт margin-top: -100dvh на сцене 2 и z-index: 200 > 100)

---

### ФАЗА 8 — Полировка и финал
**Цель:** типографика, финальные стили, SEO, production-ready

#### Задачи:
- [ ] Google Fonts — выбор и подключение через `<link>` (preconnect + stylesheet)
- [ ] `css/tokens.css` — итоговая типографика, цветовая схема
- [ ] `font-display: block` на критическом шрифте — обязательно:
  ```css
  /* font-display: swap вызывает layout shift после document.fonts.load(),
     что даёт неверные sceneHeights если swap случится после шага 7 lifecycle */
  ```
- [ ] SEO: `<title>`, `<meta name="description">`, `<meta name="viewport">`, OG-теги
- [ ] Favicon
- [ ] Проверка LCP (сцена 1 — eager + fetchpriority="high")
- [ ] Проверка на реальном iPhone (Safari) — sticky, clip-path, карусель
- [ ] Проверка на Android Chrome — touchmove passive, carousel loop
- [ ] Lighthouse Mobile audit

---

## Lifecycle инициализации (v4.3 — финальный)
```
main()
│
├── 1. fetchContent()                              ← try/catch: сеть, HTTP, JSON
├── 2. validateContent(content, SCENES)            ← бросает ошибку при несоответствии
├── 3. renderScenes(content, SCENES)               ← DOM всех сцен
├── 4. renderFaq(content.faq)                      ← DOM в <section id="faq"> (вне сцен)
├── 5. buildSceneElementsMap()                     ← кэш в sceneElements Map
├── 6. initImageLoading()                          ← сцена 1 eager, 2–6 IntersectionObserver
├── 7. await document.fonts.load(`1em ${CRITICAL_FONT}`).catch(() => {})
│      ← КРИТИЧНО: до расчёта высот
│      ← .catch(() => {}) — не роняем lifecycle если шрифт недоступен
├── 8. recalcSceneHeights()                        ← sceneHeights[] массив
├── 9. scene.style.height = sceneHeights[i]        ← КРИТИЧНО: скролл без этого не работает
├── 10. gsap.set(".scene__mask", ...)              ← начальное состояние масок
├── 11. if (prefersReducedMotion) { ... }          ← статичный показ, else initScenes()
├── 12. initCarousel(content.carousel.slides)      ← карусель в сцене 5
├── 13. initFaq(content.faq)                       ← аккордеон в <section id="faq">
├── 14. initResizeHandler()                        ← debounce 250ms
└── 15. ScrollTrigger.refresh()                    ← финальный пересчёт позиций
```

---

## Конфигурация SCENES — эталон
```js
const SCRUB                = 0.8;
const SPLIT_END            = 0.75;  // прогресс сцены когда clip-path полностью открыт
const TEXT_START           = 0.65;  // прогресс когда первый текстблок начинает движение
                                    // overlap = SPLIT_END - TEXT_START = 0.10 (намеренно)
const MAX_TEXT_LINES       = 10;
const RESIZE_DEBOUNCE      = 250;
const IMAGE_PRELOAD_MARGIN = '200%';

const SCENES = [
  { id: 1, textBlocks: 0, isIntro: true },
  { id: 2, textBlocks: 2 },
  { id: 3, textBlocks: 1 },
  { id: 4, textBlocks: 2 },
  { id: 5, textBlocks: 1, hasCarousel: true },
  { id: 6, textBlocks: 1, hasButtons: true },
];
```

---

## Критические запреты (выжимка)

| Запрет | Причина |
|---|---|
| `position: absolute` на `.scene__mask` | Маска уезжает с потоком документа — эффект шторки невозможен, между картинками дыры |
| `transform: translateY(-50%)` на img | Растяжка вертикальных изображений на широких экранах — использовать `object-fit: cover` |
| Отсутствие `margin-top: -100dvh` на `.scene ~ .scene` | Сцены не перекрываются физически — z-index бессмысленен |
| Отсутствие `overflow: hidden` на `.scene__mask` | img (absolute, inset: 0) вылезает за границы 100dvh sticky-контейнера |
| `overflow: hidden` на `.scene`, `body`, `html` | Убивает `position: sticky` на `.scene__mask` и `.scene__title-layer` (особенно Safari) |
| `contain: layout style` на `.scene` | Ломает `sticky` и `scrollHeight` в Safari |
| `transform / filter` на `.scene` | Создаёт stacking context, ломает `sticky` |
| `clip-path` на `.scene` (не на `.scene__mask`) | Ломает `sticky` на iOS Safari |
| `.scene__image-layer` как отдельный слой | Лишняя вложенность — img прямо в `.scene__mask` |
| `duration` в px в timeline | Разрушает нормализованную модель |
| `scrub: true` или `scrub: 1` | Перескакивает фазы / слишком долгая инерция |
| `passive: true` на `touchmove` карусели | Блокирует `preventDefault` |
| Swiper | Конфликт с touch-action логикой |
| Вызов `recalcSceneHeights()` до `document.fonts.load()` | Высоты будут неверными |
| Установка `style.height` без `recalcSceneHeights()` | Скролл не работает |
| `BASE_VH` как CSS-строка `'100vh'` | Jank на мобильных при скрытии адресной строки браузера |
| `opacity` для анимации `.scene__text-block` через scrub | Только `transform: translateY` — opacity мигает при скрабе |
| GSAP-анимация для кнопок сцены 6 | Кнопки — статичные прямоугольники, никакой track/scaleX/label анимации |
| `.scene__mask` или sticky-элементы в `.scene--intro` | Интро — статичная сцена, никакой scroll-механики |
| `style.height` через JS для сцены 1 | Высота интро фиксирована CSS: `height: 100dvh` |
| `data-src` на img сцены 1 | Картинка интро грузится eager немедленно, не через Observer |
| `will-change` статически в CSS | Только динамически через JS; снимать через `onLeave`/`onLeaveBack` |
| Магические числа в `engine.js` для порогов анимации | Все пороги — именованные константы из `config.js` |
| `hasFaq: true` в конфиге сцены | FAQ — отдельная секция вне сцен |
| `ScrollTrigger.refresh()` в `initFaq()` | FAQ вне зон триггеров — refresh не нужен |
| Отдельный GSAP для заголовков | `position: sticky` даёт идентичный эффект бесплатно |

---

## Инварианты (должны выполняться всегда)

- ✔ Один animation timeline + один will-change ScrollTrigger на сцену
- ✔ `.scene ~ .scene` имеет `margin-top: -100dvh` — физическое перекрытие сцен
- ✔ `.scene__mask` имеет `position: sticky; top: 0` — залипание картинки на viewport
- ✔ `overflow: hidden` на `.scene__mask` — img не вылезает за пределы sticky-контейнера
- ✔ `object-fit: cover` на `.scene__mask img` — пропорции без растяжки при любом соотношении сторон
- ✔ img живёт прямо в `.scene__mask` — отдельный `.scene__image-layer` отсутствует
- ✔ Timeline нормализован через `SPLIT_END`, `TEXT_START`, `blockRatio` из `config.js` (без магических чисел)
- ✔ Overlap split↔text через `overlapRatio = SPLIT_END - TEXT_START`
- ✔ Триггер nth текстблока: старт когда верхний край (n-1)-го блока уходит за top: 0
- ✔ "Буфер чтения" в конце каждой сцены — ожидаемое поведение, не баг
- ✔ Все анимации обратимы (scroll вверх = точная инверсия)
- ✔ Заголовок движется за сплит-линией через `position: sticky` — отдельный GSAP не добавлять
- ✔ DOM-элементы сцен кэшированы в `sceneElements Map`
- ✔ Контент только из `content.json`
- ✔ `validateContent()` до рендера
- ✔ `style.height` явно после каждого `recalcSceneHeights()`
- ✔ `ScrollTrigger.refresh()` в конце `main()` — один раз
- ✔ `BASE_VH` в `initScene` — `window.innerHeight` в пикселях, зафиксированный при инициализации
- ✔ FAQ — отдельная `<section id="faq">` после `<main id="scenes-container">`, вне сцен
- ✔ `will-change` снимается через `onLeave`/`onLeaveBack` — нет утечки GPU
- ✔ Два независимых IntersectionObserver: вертикальный (images.js) и горизонтальный (carousel.js)
- ✔ Сцена 1 (`isIntro: true`) — статичная, `height: 100dvh` в CSS, пропускается `initScenes()`, img eager без data-src
- ✔ Кнопки сцены 6 — статичные горизонтальные прямоугольники, без GSAP-анимации
- ✔ `fetchContent()` оборачивает fetch в try/catch — падение не роняет страницу молча
- ✔ Шрифт подключён с `font-display: block` — нет layout shift после `fonts.load()`

---

## ЗОНЫ РИСКА — статус и комментарии

### ✅ Зашиты в план (применены)

| Риск | Решение |
|---|---|
| **Р1: `contain: layout style`** | Убран из CSS `.scene` — не использовать |
| **Р2: "Буфер" / пустой хвост timeline** | Задокументировано как ожидаемое поведение ("зона чтения") |
| **Р5: `prefers-reduced-motion`** | Расширен блок: `gsap.set` для mask + textBlocks + buttons/labels |
| **Р6: ScrollTrigger.refresh()** | Добавлен в конец `main()` как шаг 15 |
| **Р8: "Прыгающий" скролл при открытии FAQ** | Закрыт автоматически — FAQ вынесен из зоны ScrollTrigger |
| **Р9: Высота маски [FIX v4.3]** | `.scene__mask { position: sticky; height: 100dvh; overflow: hidden }` |
| **Р10: BASE_VH как строка [FIX v4.3]** | `const BASE_VH = window.innerHeight` — фиксированные пиксели при init |
| **Р11: sticky + Safari [NEW v4.3]** | Задокументировано: нет `overflow: hidden` на `.scene`, `body`, `html` |

### ⚠️ Мониторинг в процессе (не ломают план, но требуют внимания при тестировании)

**Риск 3: `document.fonts.load()` — не гарантирует все шрифты**
- Когда: фаза 3 (расчёт высот), фаза 8 (тест на реальных устройствах)
- Симптом: лёгкий сдвиг текста после инициализации
- Решение в плане: `document.fonts.load(`1em ${CRITICAL_FONT}`)` для конкретного шрифта + `font-display: block`
- `.catch(() => {})` — lifecycle не падает если шрифт недоступен (офлайн)
- При bugs: добавить `requestAnimationFrame` задержку перед `recalcSceneHeights()`

**Риск 4: Рассинхрон двойных триггеров (timeline + will-change)**
- Когда: фаза 3 (инициализация engine), фаза 6 (resize)
- Симптом: `will-change` активирован, анимация ещё не началась — GPU-слои "горят" впустую
- Компенсация: зоны заданы корректно (`start: "top 120%"` / `end: "bottom -20%"`)
- `onLeave`/`onLeaveBack` снимают `will-change: auto` — утечка GPU закрыта
- При bugs: расширить зону до `start: "top 150%"`

**Риск 7: Переполнение заголовка (Title Overflow)**
- Когда: фаза 1 (валидация), фаза 2 (CSS)
- Симптом: длинный заголовок перекрывает первый текстблок
- Компенсация: `console.warn()` в `validateContent()` при заголовке > 3 строк
- При bugs CSS: добавить `max-height` + `overflow: hidden` на `.scene__title-layer`
