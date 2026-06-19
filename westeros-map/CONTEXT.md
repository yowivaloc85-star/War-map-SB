# Westeros War Map — Полный контекст проекта

> **Дата последнего обновления:** 2026-05-17  
> **Файл проекта:** `C:\Users\bruh\westeros-map\index.html` — один файл, всё внутри  
> **Резервная копия:** `C:\Users\bruh\Desktop\westeros-map-backup.html`  
> **Запуск локально:** `python -m http.server 5500 --directory C:\Users\bruh\westeros-map` → `http://localhost:5500`  
> Или открыть `index.html` напрямую в браузере (двойной клик / перетащить).

---

## 1. Архитектура

### Общая структура страницы
```
<html>
  <head> CSS (всё в одном блоке <style>) </head>
  <body>
    .header (лого + заголовок)
    #map-wrap (левая часть, вся ширина минус 300px справа)
      #svg-container
        <svg viewBox="..."> ← единственная точка зума/пана
          <defs> (паттерны сетки, minor/major grid) </defs>
          <g id="ocean"> (фон — rect) </g>
          <g id="land" class="land"> (все регионы SVG paths) </g>
          <g id="rivers"> </g>
          <g id="lakes"> </g>
          <g id="borders"> (стена, границы регионов) </g>
          <g id="region-labels"> (названия регионов, drag) </g>
          <g id="tiles"> </g>         ← тайлы
          <g id="roads"> </g>         ← дороги
          <g id="guides"> (сетка) </g>
          [города рендерятся в отдельный <g> через renderCities]
          [рельеф — в отдельный <g> через renderTerrain]
          [drag-ep-handles — ручки конечных точек дорог]
          [road-preview-g — резиновая нить при рисовании дороги]
          [previewG тайлов — preview при рисовании тайла]
        </svg>
    #info (правая панель, 300px, инфо о выбранном регионе/городе/тайле)
    [admin-btn, pw-modal, admin-panel — вставляются через insertAdjacentHTML]
    [pick-hint, tile-draw-hint, road-draw-hint — хинты снизу по центру]
    [drag-mode-btn — кнопка ✥]
    [sync-badge — значок Firebase онлайн/офлайн]
  </body>
  <script> (весь JS) </script>
  <script Firebase SDK>
  <script> Firebase Sync IIFE </script>
</html>
```

### Структура JS (порядок в файле)
```
1. regionData (константа, описания регионов)
2. showInfo(id) — отобразить регион в #info
3. Pan & Zoom: SVG_W=3080, SVG_H=4740, vx/vy/vw/vh, applyViewBox(), wheel, drag, dblclick-reset
4. Клики по регионам (namedRegions → .clickable)
5. Region labels IIFE — renderLabels(), drag labels, labelOffsets
6. Cities IIFE — citiesData[], renderCities(), updateCities()
7. Terrain IIFE — terrainData[], renderTerrain()
8. Roads IIFE — roadsData[], renderRoads(), setRoadEditMode(), updateRoadsVisibility()
9. Tiles IIFE — tilesData[], renderTiles(), draw tools, snap, updateTilesVisibility()
10. window._persistMap() — единая точка сохранения в localStorage
11. Admin Panel IIFE (insertAdjacentHTML) — пароль, вкладки, редакторы
12. Drag Mode IIFE — ✥ кнопка, перетаскивание городов/рельефа/дорог/меток
13. Firebase Sync IIFE — переопределяет _persistMap, realtime onValue
```

---

## 2. Pan & Zoom

```javascript
const SVG_W = 3080, SVG_H = 4740;
let vx = 0, vy = 0, vw = SVG_W, vh = SVG_H;

function getScale() { return Math.max(mapWrap.offsetWidth/vw, mapWrap.offsetHeight/vh); }

// SVG координаты из mouse event:
const r  = mapWrap.getBoundingClientRect();
const sc = getScale();
const svgX = Math.round(vx + (e.clientX - r.left) / sc);
const svgY = Math.round(vy + (e.clientY - r.top)  / sc);

// applyViewBox вызывается из: колёсика, pan mousemove, dblclick-reset
function applyViewBox() {
  vw = clamp(vw, 200, SVG_W);
  vh = vw * (SVG_H / SVG_W);
  // ... clamp vx/vy ...
  svg.setAttribute('viewBox', `${vx} ${vy} ${vw} ${vh}`);
  updateLabels();
  updateCities();
  if (window.updateTilesVisibility) window.updateTilesVisibility();
  if (window.updateRoadsVisibility) window.updateRoadsVisibility();
}
```

**Пан:** ЛКМ в обычном режиме. ПКМ или СКМ в любом режиме редактирования.  
**Зум:** колёсико мыши с центром на курсоре.  
**Сброс:** двойной клик на карте.

---

## 3. Пороги видимости слоёв (zoom-based fade)

| Слой | Порог `vw` | Логика |
|---|---|---|
| Названия регионов | `vw >= SVG_W * 0.85` (~2620) | видны только при отдалении |
| Города | `vw <= SVG_W / 2` (≤1540) | видны при приближении 2× |
| Тайлы | `vw <= SVG_W * 0.45` (≤1386) | видны при приближении ~2.2× |
| Дороги | `vw <= SVG_W * 0.45` (≤1386) | то же; всегда видны при active edit mode |

Все слои используют CSS `transition: opacity 0.35s` + JS `element.style.opacity`.  
**Не `display:none`** — только `opacity` + `pointerEvents`.

---

## 4. Данные и сохранение

### Форматы данных
```javascript
// Города
window.citiesData = [
  ['Название', x, y, 'desc', 'image.jpg'],
  // ...
];

// Рельеф
window.terrainData = [
  [typeInt, x, y, scale],
  // typeInt: 0=горы, 1=холмы, 2=лес, 3=болото, 4=пустыня, 5=снег
];

// Дороги
window.roadsData = [
  { id: 'road-...', name: 'Название', points: [[x,y], [x,y], ...] }
];

// Тайлы
window.tilesData = [
  { id: 'tile-...', name: '', desc: '', color: '#4a8a5a', points: [[x,y],...] }
];

// Смещения меток регионов
window.labelOffsets = {
  'dorne': { dx: -320, dy: -480 },
  'riverlands': { dx: -160, dy: 130 }
};
```

### localStorage ключи
```
wm_cities, wm_terrain, wm_roads, wm_tiles, wm_label_offsets
```

### `_persistMap`
```javascript
window._persistMap = function () {
  try {
    localStorage.setItem('wm_cities',        JSON.stringify(window.citiesData));
    localStorage.setItem('wm_terrain',       JSON.stringify(window.terrainData));
    localStorage.setItem('wm_roads',         JSON.stringify(window.roadsData));
    localStorage.setItem('wm_tiles',         JSON.stringify(window.tilesData));
    localStorage.setItem('wm_label_offsets', JSON.stringify(window.labelOffsets));
  } catch(e) {}
};
// Firebase Sync IIFE переопределяет эту функцию, добавляя запись в Firebase
```

---

## 5. Режимы редактирования

Все режимы добавляют CSS-класс на `body` → `#admin-panel { display: none }` (полностью скрыт).  
Во всех режимах редактирования ПКМ/СКМ панорамирует карту.

| Режим | CSS класс | Флаг | Описание |
|---|---|---|---|
| Pick точки | `pick-mode` | `pickMode = true` | клик на карту → координаты в поле |
| Рисование тайла | `tile-draw-mode` | `window._tileDrawMode` | полигон/прямоугольник/от руки |
| Рисование дороги | `road-draw-mode` | `window._roadDrawMode` | непрерывные точки, двойной клик = финиш |
| Редактирование дороги | — | `window._roadEditMode = id` | drag точек, клик на путь = вставить точку |
| Drag mode | — | `window._dragMode` | ✥ кнопка, перетаскивание всего |

```javascript
// Логика пана — учитывает все режимы:
const editing = window._tileDrawMode || window._roadDrawMode ||
                document.body.classList.contains('pick-mode') ||
                window._roadEditMode;
if (editing) {
  if (e.button !== 1 && e.button !== 2) return; // только ПКМ/СКМ
} else {
  if (e.button !== 0) return; // только ЛКМ
}
```

---

## 6. Тайлы — инструменты рисования

### Три инструмента (кнопки в табе «Тайлы»)
- **◆ Полигон** — клик за кликом добавляет вершины; закрыть на первую вершину (snap-ring) / двойной клик / Enter
- **▬ Прямоугольник** — клик первый угол, клик второй угол
- **✏ От руки** — зажать ЛКМ и рисовать; отпустить = финиш

### Упрощение (от руки)
```javascript
// Douglas-Peucker, epsilon=10 SVG units
function simplify(pts, epsilon) { ... }
function perpDist(p, a, b) { ... }
// вызов: var pts = simplify(_freehandPoints, 10);
```

### Привязка к краям (только режим «от руки»)
```javascript
var EDGE_SNAP_THRESHOLD = 38; // SVG units
var COAST_SAMPLE_STEP   = 14; // SVG units

// Ленивая инициализация — сэмплирует все <path> в #land:
function buildCoastlinePts() {
  document.querySelectorAll('#land path, #land polygon').forEach(el => {
    const len = el.getTotalLength();
    for (let t = 0; t < len; t += COAST_SAMPLE_STEP) {
      const p = el.getPointAtLength(t);
      _coastlinePts.push([p.x, p.y]);
    }
  });
}

function snapToEdges(x, y) {
  // 1. Snap к рёбрам существующих тайлов (точный сегмент)
  // 2. Snap к сэмплам береговой линии / границам регионов
  // Возвращает { pt: [x,y], snapped: bool }
}
```
При привязке — золотое кольцо на курсоре.

### Undo
`Ctrl+Z` — удаляет последний тайл из `tilesData`. Не работает во время рисования.

---

## 7. Дороги

### Рисование (кнопка «✏ Рисовать» в табе «Дороги»)
- Каждый ЛКМ-клик добавляет точку в конец дороги
- Резиновая нить (gold dashed) от последней точки к курсору
- Двойной клик / Enter — финиш (убирает дублированную точку)
- Esc — выход без удаления добавленных точек

### Редактирование
- «✏ Редактировать точки» → drag точек, двойной клик на точку = удалить её
- Клик на дорогу (в режиме редактирования) — вставить точку между ближайшими сегментами
- «📍 Точка» — одиночный pick-клик, добавляет точку в конец

### Рендер
```javascript
// SVG path через pointsToD: "M x,y L x,y L x,y ..."
// outer path (strokeWidth 14, pointer-events:stroke) — для клика/вставки
// inner path (strokeWidth 7, pointer-events:none) — визуал
```

---

## 8. Drag Mode (✥)

```javascript
function findTarget(target) {
  // data-drag-road  → { type:'road', roadId, ptIdx }
  // data-city-idx   → { type:'city', idx }
  // data-terrain-idx → { type:'terrain', idx }
  // data-label-id   → { type:'label', labelId }
}
// Обработчики на document (capture: true для mousedown)
// Сохранение: _persistMap() на mouseup
// Метки: сохраняются в window.labelOffsets[labelId] = {dx, dy}
```

Метки регионов обёрнуты в `<g data-label-id="id" transform="translate(dx,dy)">`.  
`setDragMode(on)` включает/выключает `pointer-events` на `#region-labels`.

---

## 9. Отображение городов

```javascript
// Текстовый лейбл — один <text> с paint-order:stroke (fix артефакта рендера):
const t = document.createElementNS(NS, 'text');
t.setAttribute('fill', '#f0e0c0');
t.setAttribute('stroke', '#2a0a0a');
t.setAttribute('stroke-width', '5');
t.setAttribute('stroke-linejoin', 'round');
t.setAttribute('paint-order', 'stroke'); // stroke рендерится ПОД fill
```

Города инициализируются с `opacity: 0` (не `display:none` — иначе не появляются).

---

## 10. Подсветка королевств

Каждый регион подсвечивается своим цветом через CSS `filter: brightness() sepia() saturate() hue-rotate()`.  
Hover мягкий, active насыщеннее. Переход `0.35s ease`.

| Регион | Цвет | hue-rotate |
|---|---|---|
| beyond-the-wall | ледяной голубой | 175deg |
| north | серо-стальной (Старки) | 185deg |
| iron-islands | холодный серый (Грейджой) | grayscale(0.7) |
| westerlands | ланнистерский алый | 322deg |
| riverlands | голубой (Талли) | 163deg |
| vale | небесно-голубой (Аррен) | 188deg |
| crownlands | таргариенский красный | 330deg |
| stormlands | янтарно-жёлтый | 28deg |
| reach | мягкий зелёный (Тиреллы) | 95deg |
| dorne | оранжево-красный (Мартеллы) | 348deg |

---

## 11. Admin Panel

**Пароль:** `mz?CZ{@z6YR}`  
Кнопка ⚙ в правом нижнем углу.

### Вкладки
1. **Регионы** — редактировать описание, герб, лидера, климат
2. **Города** — добавить/редактировать/удалить, pick-позиция на карте
3. **Рельеф** — добавить символ рельефа, pick-позиция
4. **Дороги** — создать, рисовать непрерывно, редактировать точки
5. **Тайлы** — нарисовать (3 инструмента), задать цвет, название, описание

### Глобальные функции, используемые admin'ом
```javascript
window.renderCities()   window.renderTerrain()
window.renderRoads()    window.renderTiles()
window.setRoadEditMode(id | null)
window._startTileDraw(color)
window._tileDrawExited = cancelTileDrawAdmin  // callback
window._tileAdminSelect(idx)                  // callback после финиша тайла
window._tileAdminOpen   // bool, следить за вкладкой
window._roadDrawMode    // bool
window._roadEditMode    // id | null
```

---

## 12. Firebase Realtime Sync

Расположен в отдельном `<script>` в конце файла, после SDK.

```javascript
// Конфиг (нужно заполнить):
var firebaseConfig = {
  apiKey: "", authDomain: "", databaseURL: "", ...
};

// Путь в БД: 'westerosMap'
// Поля: cities, terrain, roads, tiles, label_offsets

// Логика:
// - _persistMap переопределён: пишет в localStorage И в Firebase
// - onValue: при первом вызове — загружает Firebase данные (приоритет над localStorage)
//            при последующих — сравнивает JSON, если изменилось — rerenderAll()
// - _writing флаг (+ 1.5s таймер) предотвращает рендер от собственного эха
// - значок ● синхронизация / ○ офлайн (позиция: bottom:60px right:16px)
```

**Правила Firebase (тестовый режим):**
```json
{ "rules": { ".read": true, ".write": true } }
```

---

## 13. Деплой

**GitHub Pages:**
```bash
cd C:\Users\bruh\westeros-map
git init
git add .
git commit -m "westeros map"
git branch -M main
git remote add origin https://github.com/USERNAME/westeros-map.git
git push -u origin main
# Settings → Pages → Branch: main → Save
# URL: https://USERNAME.github.io/westeros-map
```

**Netlify Drop:** перетащить папку на app.netlify.com/drop.

**Обновление после изменений:**
```bash
git add index.html && git commit -m "update" && git push
```

---

## 14. Ключевые технические решения

| Проблема | Решение |
|---|---|
| Зум без blur | Манипуляция `viewBox` напрямую, без CSS `transform` |
| Порядок событий (draw mode перехватывает клики) | `addEventListener(..., true)` — capture phase |
| Артефакт рендера текста (красный «н») | Один `<text>` с `paint-order: stroke` вместо двух слоёв |
| Города переставали появляться | `opacity: 0` при инициализации вместо `display: none` |
| Плавная видимость слоёв | `opacity` + CSS `transition`, не `display:none` |
| Перетаскиваемые метки регионов | `<g data-label-id transform="translate(dx,dy)">` |
| Snap к береговой линии | Ленивый сэмплинг `getTotalLength`/`getPointAtLength` всех `#land path` |
| Бесконечный цикл Firebase (эхо своих записей) | Флаг `_writing = true` на 1.5с после записи |
| Пан в режиме рисования | ПКМ/СКМ обходит capture-handler draw mode (он ловит только ЛКМ) |

---

## 15. Текущий статус и TODO

### Сделано ✓
- [x] SVG карта Вестероса с кликабельными регионами
- [x] Pan/zoom через viewBox
- [x] Инфо-панель справа (регион, город, тайл)
- [x] Координаты на hover (отображение)
- [x] Admin panel с паролем
- [x] Редактор городов (добавить/переместить/удалить/pick)
- [x] Редактор рельефа (горы, леса и т.д.)
- [x] Редактор дорог (draw continuous, edit points, drag endpoints)
- [x] Тайлы — рисование (полигон/прямоугольник/от руки)
- [x] Привязка к тайлам и береговой линии (только «от руки»)
- [x] Undo тайлов (Ctrl+Z)
- [x] Drag mode (✥) — города, рельеф, дороги, метки регионов
- [x] Zoom-based visibility: города, тайлы, дороги (0.35s fade)
- [x] Перетаскиваемые метки регионов
- [x] Подсветка регионов своими цветами (per-kingdom CSS filter)
- [x] localStorage persistence
- [x] Firebase Realtime Sync (заготовка — нужно вставить config)
- [x] Панорамирование ПКМ в любом режиме редактирования

### TODO / Возможные улучшения
- [ ] **Firebase config** — вставить реальные ключи и задеплоить
- [ ] Защита записи в Firebase (Firebase Auth для admin)
- [ ] Экспорт / Импорт карты (JSON кнопка в admin) — для передачи состояния
- [ ] Отображение координат при наведении (показывать в UI постоянно)
- [ ] Snap в режиме полигона (если понадобится)
- [ ] Редактирование существующих тайлов (изменить форму, не только цвет/название)
- [ ] Поддержка мобильных устройств (touch pan/zoom)

---

## 16. Файловая структура

```
C:\Users\bruh\westeros-map\
  index.html          ← ВСЁ (HTML + CSS + JS + SVG карта)
  arryn.jpg
  baratheon.jpg
  greyjoy.jpg
  lannister.jpg
  martell.jpg
  stark.jpg
  targaryen.jpg
  tylly.jpg
  tyrell.jpg
  [лого файл(ы)]
  CONTEXT.md          ← этот файл

C:\Users\bruh\.claude\launch.json  ← конфиг запуска dev-сервера
  { "runtimeExecutable":"python", "runtimeArgs":["-m","http.server","5500",...] }
```

---

*Документ сгенерирован как точка восстановления контекста. При вставке в новый чат Claude сразу входит в курс дела.*
