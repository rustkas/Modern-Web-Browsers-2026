# Глава 18. Интерфейсы будущего: AR/VR и жесты

В 2026 году веб-интерфейсы вышли за пределы двумерного экрана. Браузер стал платформой для иммерсивных взаимодействий — дополненной и виртуальной реальности (AR/VR) через WebXR, пространственного позиционирования, трекинга рук и жестов. Эта глава детально разбирает актуальное состояние WebXR, модели захвата рук и пространственного позиционирования в WebXR и OpenXR, а также эволюцию обработки жестов в CSS через свойства `pointer-events` и `touch-action`.

---

## 1. WebXR 2.0: захват рук и пространственное позиционирование

**WebXR Device API** — это стандарт W3C, позволяющий веб-приложениям взаимодействовать с устройствами виртуальной и дополненной реальности . В 2026 году WebXR достиг статуса Candidate Recommendation и развивается в сторону полноценной Recommendation . Он заменил устаревший WebVR, унифицировав работу с VR и AR под единым дизайном .

### 1.1. Сессионная модель WebXR

Входная точка в WebXR — `navigator.xr`. Приложение проверяет поддержку режима сессии, затем запрашивает сессию по жесту пользователя (клик/тап) :

```javascript
if (navigator.xr) {
    const supported = await navigator.xr.isSessionSupported('immersive-vr');
    if (supported) {
        button.onclick = async () => {
            const session = await navigator.xr.requestSession('immersive-vr', {
                optionalFeatures: ['local-floor', 'hand-tracking']
            });
            onSessionStarted(session);
        };
    }
}
```

**Режимы сессий** :

- **`inline`** — рендеринг в окне страницы ("магическое окно"), не требует специальных разрешений.
- **`immersive-vr`** — эксклюзивный доступ к VR-дисплею, требует разрешения.
- **`immersive-ar`** — дополненная реальность с прозрачным (see-through) отображением, требует AR-модуля и AR-совместимого устройства.

### 1.2. Пространственное позиционирование

**Пространственные пространства (reference spaces)** — фундаментальная концепция WebXR, определяющая систему координат для позиционирования виртуальных объектов :

- **`viewer`** — относительно позиции головы пользователя.
- **`local`** — относительно начальной позиции устройства (XR_SPACE_LOCAL) .
- **`local-floor`** — относительно пола в начальной позиции устройства, требует согласия на `xr-spatial-tracking` .
- **`bounded-floor`** — с ограниченной областью движения (для комнатного масштаба).

Пространства используются для **hit testing** (определения реальных поверхностей) и **anchors** (закрепления виртуальных объектов в реальном пространстве) :

```javascript
// Hit testing для AR
const viewerSpace = await session.requestReferenceSpace('viewer');
const hitTestSource = await session.requestHitTestSource({ space: viewerSpace });

// В frame loop
const results = frame.getHitTestResults(hitTestSource);
if (results.length) {
    const hitPose = results[0].getPose(refSpace);
    // Размещение объекта в hitPose.transform.matrix
}
```

### 1.3. Трекинг рук (Hand Tracking)

Трекинг рук в WebXR реализован через модуль **Hand Input**, основанный на OpenXR `XR_EXT_hand_tracking` extension .

**Архитектура трекинга рук:**

Спецификация OpenXR определяет 26 суставов руки (joints) для каждой руки :

```c
typedef enum XrHandJointEXT {
    XR_HAND_JOINT_PALM_EXT = 0,
    XR_HAND_JOINT_WRIST_EXT = 1,
    XR_HAND_JOINT_THUMB_METACARPAL_EXT = 2,
    // ... 26 суставов всего
    XR_HAND_JOINT_LITTLE_TIP_EXT = 25,
} XrHandJointEXT;
```

Схема суставов включает: 4 сустава для большого пальца, 5 суставов для каждого из остальных пальцев, запястье и ладонь .

**Получение данных о суставах в OpenXR** :

```c
// Создание трекера для левой руки
XrHandTrackerCreateInfoEXT createInfo{...};
createInfo.hand = XR_HAND_LEFT_EXT;
createInfo.handJointSet = XR_HAND_JOINT_SET_DEFAULT_EXT;
pfnCreateHandTrackerEXT(session, &createInfo, &leftHandTracker);

// Для каждого кадра
XrHandJointsLocateInfoEXT locateInfo{...};
locateInfo.baseSpace = worldSpace;
locateInfo.time = frameState.predictedDisplayTime;

pfnLocateHandJointsEXT(leftHandTracker, &locateInfo, &locations);

if (locations.isActive) {
    // Доступ к позиции и радиусу каждого сустава
    const XrPosef &indexTipPose = jointLocations[XR_HAND_JOINT_INDEX_TIP_EXT].pose;
    const float indexTipRadius = jointLocations[XR_HAND_JOINT_INDEX_TIP_EXT].radius;
}
```

Каждый сустав имеет :
- **`pose`** — позиция и ориентация сустава (6DoF).
- **`radius`** — радиус сферы аппроксимации сустава (в метрах), может использоваться для коллизий (например, нажатие виртуальной кнопки кончиком пальца) .
- **`locationFlags`** — битовые флаги, указывающие, какие поля валидны .

Для получения скоростей (линейной и угловой) используется структура `XrHandJointVelocitiesEXT` .

**Применение трекинга рук в XR-приложениях 2026 года:**

- **Прямое взаимодействие.** Нажатие виртуальных кнопок, захват объектов (one-hand, two-hand, distance grab) .
- **Жесты.** Распознавание жестов через комбинации поз суставов, смена режимов взаимодействия через `set_input_mode` (контроллеры ↔ hand tracking) .
- **Локомоция и физика.** Управление перемещением через жесты и интеграция с физическими движками .
- **Телеоперация роботов.** Передача 3D-позиций суставов рук для управления роботизированными манипуляторами .

**Практический совет:** Рекомендуется использовать названия суставов (например, `Index_Metacarpal_L`, `Thumb_Tip_R`) вместо индексов для портируемости ассетов между движками .

### 1.4. Ограничения и особенности WebXR 2026

- **AR на Safari.** visionOS Safari поддерживает `immersive-vr`, но AR-модуль пока недоступен; iOS/macOS Safari вообще не имеют WebXR .
- **Требование жеста пользователя.** `requestSession()` для иммерсивных режимов должна вызываться из жеста пользователя (клик/тап) .
- **Совместимость с WebGL.** Контекст WebGL должен быть создан как XR-совместимый (или сделан таковым) .
- **Feature detection.** Не все модули (hit test, anchors, depth, hand tracking) доступны на всех устройствах; их следует запрашивать как `optionalFeatures` и проверять поддержку во время выполнения .

---

## 2. Новая модель обработки жестов в CSS и Pointer Events

Помимо иммерсивных интерфейсов, веб-платформа 2026 года предоставляет богатые инструменты для обработки жестов на традиционных устройствах (тач-экраны, трекпады, мыши) через CSS-свойства и Pointer Events.

### 2.1. CSS pointer-events

Свойство `pointer-events` устанавливает, может ли элемент быть целью pointer-событий . В 2026 году оно широко поддерживается и используется для управления интерактивностью в сложных UI.

**Ключевые значения** :

| Значение | Применение | Описание |
|----------|------------|----------|
| **`auto`** | HTML / SVG | Элемент ведет себя по умолчанию, получает pointer-события. |
| **`none`** | HTML / SVG | Элемент никогда не является целью событий; событие "проходит сквозь" на элемент под ним . |
| **`visiblePainted`** | SVG | Элемент получает события только когда видим и курсор над залитой/обведенной областью . |
| **`fill` / `stroke`** | SVG | Элемент получает события только когда курсор над заливкой или над обводкой (независимо от значений свойств). |
| **`bounding-box`** | SVG | Элемент получает события внутри своего ограничивающего прямоугольника, независимо от формы . |

**Важное примечание:** `pointer-events: none` не препятствует событию `pointerenter` / `pointerleave` на родительском элементе при перемещении указателя над дочерними элементами, которым разрешено быть целями событий . Также элементы с `pointer-events: none` получают фокус через клавиатурную навигацию (Tab) .

### 2.2. CSS touch-action

Свойство `touch-action` управляет поведением сенсорных жестов (скролл, зум) на элементе .

В 2026 году в Chromium было реализовано важное улучшение: **поддержка single-axis scroll containers** . Ранее `pan-x` и `pan-y` всегда включались вместе в контейнерах с `touch-action: none`. Теперь `pan-x` / `pan-y` включаются **независимо** в зависимости от того, является ли элемент scroll-контейнером по данной оси .

```css
/* Элемент скроллится только по вертикали, но внутри touch-action: none */
.scroll-y-only {
    touch-action: none;  /* Отключает все жесты */
    overflow-y: scroll;  /* Контейнер скролла по Y */
    /* Chrome 146+ автоматически разрешит pan-y, но не pan-x */
}
```

Это позволяет более тонко управлять жестами в сложных UI, где элементы имеют скролл только по одной оси.

### 2.3. Pointer Events и жесты

Pointer Events унифицируют обработку мыши, пера и касания. В 2026 году они поддерживаются во всех браузерах.

**Ключевые события:** `pointerdown`, `pointerup`, `pointermove`, `pointercancel`, `pointerenter`, `pointerleave` .

**Пример обработки жестов** (с использованием `touch-action` и Pointer Events):

```css
/* Элемент не должен обрабатывать системные жесты */
.draggable {
    touch-action: none;
    user-select: none;
}
```

```javascript
let isDragging = false;

element.addEventListener('pointerdown', (e) => {
    isDragging = true;
    element.setPointerCapture(e.pointerId);
});

element.addEventListener('pointermove', (e) => {
    if (isDragging) {
        // Обработка перемещения
        element.style.transform = `translate(${e.clientX}px, ${e.clientY}px)`;
    }
});

element.addEventListener('pointerup', (e) => {
    isDragging = false;
    element.releasePointerCapture(e.pointerId);
});
```

---

## Итог

В 2026 году интерфейсы будущего — это две взаимодополняющие парадигмы:

1. **Иммерсивные интерфейсы (AR/VR).** WebXR Device API обеспечивает доступ к XR-устройствам с поддержкой пространственного позиционирования (`local`, `local-floor`, `bounded-floor`) и трекинга рук через 26 суставов . Руки могут использоваться для прямого взаимодействия, жестов и телеоперации . WebXR требует жеста пользователя для запуска и поддерживает модули для hit testing, anchors, depth sensing .

2. **Жесты на традиционных устройствах.** CSS `pointer-events` управляет способностью элементов быть целями событий (включая "прозрачные" элементы через `none` и SVG-специфичные режимы) . CSS `touch-action` контролирует системные жесты, а в 2026 году получила поддержку независимого включения `pan-x`/`pan-y` для single-axis scroll-контейнеров . Pointer Events обеспечивают унифицированную обработку мыши, пера и касания.

В Заключении книги мы подведем итоги эволюции браузера как мета-ОС и наметим дорожную карту развития на 2027 год.

---

*Перейти к Заключению*