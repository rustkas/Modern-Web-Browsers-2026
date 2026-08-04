
## 📚 Содержание

### Введение. Браузер как мета-ОС

- От рендеринга HTML к управлению GPU и нейросетями
- Почему в 2026 году знать браузер важнее, чем знать фреймворк
- Обзор экосистемы: Chromium 26, Firefox 140, Safari 18 и новый движок Ladybird

* [📖 Читать введение](./book/introduction.md)
* [📚 Литература](./references/introduction.md)
* [💻 Примеры](./examples/introduction.md)

---

# Часть I. Архитектурный фундамент

## Глава 1. Внутреннее устройство браузера нового поколения

- Многопроцессорная архитектура: Browser Process, Renderer Process, GPU Process, Network Process
- Site Isolation и защита от Spectre/Meltdown на аппаратном уровне
- Работа с виртуальной памятью и сборщиком мусора (Oilpan в Blink)

* [📖 Читать главу](./book/chapter-01.md)
* [📚 Литература](./references/chapter-01.md)
* [💻 Примеры](./examples/chapter-01.md)
* [🧪 Практика](./exercises/chapter-01.md)

---

## Глава 2. Потоки, параллелизм и шедулинг

- Event Loop 2026: Microtasks, MacroTasks и Prioritized Task Scheduling API
- Web Workers и Shared Workers: когда выгружать тяжелые вычисления
- Новый Scheduler API: контроль приоритетов задач в реальном времени

* [📖 Читать главу](./book/chapter-02.md)
* [📚 Литература](./references/chapter-02.md)
* [💻 Примеры](./examples/chapter-02.md)
* [🧪 Практика](./exercises/chapter-02.md)

---

## Глава 3. Рендеринг и композитинг без компромиссов

- Пайплайн CSS/HTML: от DOM-дерева до экранных пикселей
- Новый Layout Engine (Flexbox 2026, Grid 3.0, Masonry)
- Offscreen Rendering и как браузер экономит батарею

* [📖 Читать главу](./book/chapter-03.md)
* [📚 Литература](./references/chapter-03.md)
* [💻 Примеры](./examples/chapter-03.md)
* [🧪 Практика](./exercises/chapter-03.md)

---

# Часть II. Системные API: Работа с данными и файловой системой

## Глава 4. Хранение данных за пределами localStorage

- IndexedDB 3.0 и SQLite в браузере (OPFS)
- File System Access API: работа с нативной файловой системой как с локальной папкой
- Хранилище и квоты: как не переполнить диск пользователя

* [📖 Читать главу](./book/chapter-04.md)
* [📚 Литература](./references/chapter-04.md)
* [💻 Примеры](./examples/chapter-04.md)
* [🧪 Практика](./exercises/chapter-04.md)

---

## Глава 5. Стриминг и обработка больших данных

- Streams API: Readable, Writable, TransformStream для данных терабайтных размеров
- Backpressure (противодавление) и работа с сетевыми потоками
- Fetch + Streams: рендеринг HTML по частям (Streaming SSR)

* [📖 Читать главу](./book/chapter-05.md)
* [📚 Литература](./references/chapter-05.md)
* [💻 Примеры](./examples/chapter-05.md)
* [🧪 Практика](./exercises/chapter-05.md)

---

# Часть III. Графика, ИИ и Вычисления (Hardware Acceleration)

## Глава 6. WebGPU — графический движок будущего

- Переход с WebGL на WebGPU: архитектура и безопасность
- Вычисления на GPU (GPGPU) для физических симуляций и обработки изображений
- Работа с буферами и шейдерами (WGSL)

* [📖 Читать главу](./book/chapter-06.md)
* [📚 Литература](./references/chapter-06.md)
* [💻 Примеры](./examples/chapter-06.md)
* [🧪 Практика](./exercises/chapter-06.md)

---

## Глава 7. WebNN (Web Neural Network) — ИИ в каждом браузере

- Использование NPU/TPU через стандарт WebNN API
- Запуск моделей (ONNX, TensorFlow Lite) без драйверов
- Сценарии: ресайз изображений, удаление шума, перевод речи в реальном времени

* [📖 Читать главу](./book/chapter-07.md)
* [📚 Литература](./references/chapter-07.md)
* [💻 Примеры](./examples/chapter-07.md)
* [🧪 Практика](./exercises/chapter-07.md)

---

## Глава 8. WebAssembly (WASM) 2.0 и Component Model

- WASI (System Interface) — запуск нативных приложений в песочнице
- Многопоточность в WASM и совместное использование памяти с JS
- Портирование тяжелых десктопных приложений (Figma, AutoCAD) в браузер

* [📖 Читать главу](./book/chapter-08.md)
* [📚 Литература](./references/chapter-08.md)
* [💻 Примеры](./examples/chapter-08.md)
* [🧪 Практика](./exercises/chapter-08.md)

---

# Часть IV. Сеть, Навигация и Service Workers

## Глава 9. Современная сетевая модель (HTTP/3 и QUIC)

- Fetch API 2026: новый контроллер сигналов и таймаутов
- HTTP/3 и 0-RTT: как браузер ускоряет соединения
- Network Information API: адаптация под медленные сети

* [📖 Читать главу](./book/chapter-09.md)
* [📚 Литература](./references/chapter-09.md)
* [💻 Примеры](./examples/chapter-09.md)
* [🧪 Практика](./exercises/chapter-09.md)

---

## Глава 10. Service Workers как прокси-сервер

- Стратегии кеширования нового поколения (Stale-While-Revalidate + Background Sync)
- Прогрессивные Web-приложения (PWA) как обычные приложения ОС
- Push API и уведомления с высоким приоритетом

* [📖 Читать главу](./book/chapter-10.md)
* [📚 Литература](./references/chapter-10.md)
* [💻 Примеры](./examples/chapter-10.md)
* [🧪 Практика](./exercises/chapter-10.md)

---

## Глава 11. Navigation API и SPA-маршрутизация без боли

- Замена старого History API на декларативную навигацию
- Intercepting навигации: префетчинг и предзагрузка ресурсов
- Scroll Restoration и управление состояниями переходов

* [📖 Читать главу](./book/chapter-11.md)
* [📚 Литература](./references/chapter-11.md)
* [💻 Примеры](./examples/chapter-11.md)
* [🧪 Практика](./exercises/chapter-11.md)

---

# Часть V. Безопасность и Приватность

## Глава 12. Модель изоляции и Permissions API

- Тонкая настройка прав (геолокация, камера, микрофон, уведомления)
- Trust Tokens и Private State Tokens — борьба с фродом без куков
- Federated Credential Management (FedCM) — вход без сторонних куков

* [📖 Читать главу](./book/chapter-12.md)
* [📚 Литература](./references/chapter-12.md)
* [💻 Примеры](./examples/chapter-12.md)
* [🧪 Практика](./exercises/chapter-12.md)

---

## Глава 13. Защита от цифровых отпечатков (Fingerprinting)

- Новые заголовки клиентских подсказок (Client Hints)
- Как работает Storage Partitioning и UA-CH
- Баланс между аналитикой и приватностью пользователя

* [📖 Читать главу](./book/chapter-13.md)
* [📚 Литература](./references/chapter-13.md)
* [💻 Примеры](./examples/chapter-13.md)
* [🧪 Практика](./exercises/chapter-13.md)

---

# Часть VI. Отладка, Инструменты и Оптимизация

## Глава 14. Chrome DevTools 2026: Профилирование системного уровня

- Панель Performance: анализ активности потоков GPU и Worker'ов
- Панель Memory: поиск утечек в SharedArrayBuffers
- Recorder нового поколения: автоматизация тестирования производительности

* [📖 Читать главу](./book/chapter-14.md)
* [📚 Литература](./references/chapter-14.md)
* [💻 Примеры](./examples/chapter-14.md)
* [🧪 Практика](./exercises/chapter-14.md)

---

## Глава 15. Core Web Vitals и метрики пользователя

- LCP, INP, CLS — как браузер измеряет счастье пользователя
- INP (Interaction to Next Paint) — главная метрика 2026
- Использование Performance Observer и User Timings

* [📖 Читать главу](./book/chapter-15.md)
* [📚 Литература](./references/chapter-15.md)
* [💻 Примеры](./examples/chapter-15.md)
* [🧪 Практика](./exercises/chapter-15.md)

---

## Глава 16. Сборка и доставка кода в эпоху HTTP/3

- ES Modules и Import Maps 2026
- Динамический import и Code Splitting на уровне браузера (Speculation Rules API)
- Рендеринг на стороне сервера (SSR) vs Рендеринг на клиенте (CSR): гибридный подход

* [📖 Читать главу](./book/chapter-16.md)
* [📚 Литература](./references/chapter-16.md)
* [💻 Примеры](./examples/chapter-16.md)
* [🧪 Практика](./exercises/chapter-16.md)

---

# Часть VII. Будущее и Экспериментальные технологии

## Глава 17. WebTransport и WebCodecs

- Замена WebSocket на WebTransport (мультиплексирование, низкая задержка)
- Кодирование/декодирование видео и аудио на лету (WebCodecs API)

* [📖 Читать главу](./book/chapter-17.md)
* [📚 Литература](./references/chapter-17.md)
* [💻 Примеры](./examples/chapter-17.md)
* [🧪 Практика](./exercises/chapter-17.md)

---

## Глава 18. Интерфейсы будущего: AR/VR и жесты

- WebXR 2.0: захват рук и пространственное позиционирование
- Новая модель обработки жестов в CSS и Pointer Events

* [📖 Читать главу](./book/chapter-18.md)
* [📚 Литература](./references/chapter-18.md)
* [💻 Примеры](./examples/chapter-18.md)
* [🧪 Практика](./exercises/chapter-18.md)

---

### Заключение. Как оставаться экспертом в быстро меняющемся мире

- Карта развития стандартов W3C и WHATWG на 2027 год
- Методология изучения новых API: чтение спецификаций, тестирование в Origin Trials
- Сообщество и вклад в open-source движков

* [📖 Читать заключение](./book/conclusion.md)
* [📚 Литература](./references/conclusion.md)

---

### Приложения

#### Приложение А: Шпаргалка по всем Web API 2026

- Сетевые API (Fetch, WebTransport, WebSocket, SSE)
- API аутентификации и безопасности (WebAuthn, FedCM, Trust Tokens)
- API работы с потоками и параллелизмом (Workers, Scheduler, Atomics)
- Графика и медиа (WebGPU, WebGL, WebCodecs, WebXR)
- Хранение данных (OPFS, IndexedDB, CacheStorage, FS Access)
- ИИ и вычисления (WebNN, WASM, Web Audio)
- API устройств (Geolocation, HID, Bluetooth, Serial)
- Производительность (Performance API, Observers, Reporting)

* [📖 Читать приложение А](./appendix/appendix-a.md)

---

#### Приложение Б: Глоссарий терминов

- Архитектура (Browser Process, Renderer Process, Site Isolation)
- Рендеринг (Layout, Paint, Compositing, Tiling)
- Сеть и безопасность (CORS, HTTP/3, QUIC, CSP)
- Движки (V8, TurboFan, Ignition, JIT, WASI)
- Метрики (Core Web Vitals, LCP, INP, CLS)
- Инструменты (DevTools, Recorder, Source Maps, Origin Trials)

* [📖 Читать приложение Б](./appendix/appendix-b.md)

---

## 🔗 Полезные ресурсы

- [MDN Web Docs](https://developer.mozilla.org/) — главная документация по Web API
- [WHATWG Living Standard](https://html.spec.whatwg.org/) — актуальная спецификация HTML
- [Web Platform Status](https://webplatformstatus.com/) — статус поддержки API в браузерах
- [Chrome Platform Status](https://chromestatus.com/) — дорожная карта Chrome
- [Web.dev](https://web.dev/) — статьи по производительности и современным API
- [Can I Use](https://caniuse.com/) — таблица поддержки API в браузерах

---

## 📄 Лицензия

Книга распространяется под лицензией [Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/).

---

## 🤝 Вклад в книгу

Приветствуются:

- исправления ошибок и опечаток;
- дополнения примеров кода;
- предложения по улучшению структуры;
- переводы на другие языки.

Создавайте Issue или Pull Request в репозитории книги.

---

**Браузер — это новая операционная система. Научитесь управлять ей.**
