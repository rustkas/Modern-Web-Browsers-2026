# Часть II. Системные API: Работа с данными и файловой системой
## Глава 4. Хранение данных за пределами localStorage

В 2026 году возможности браузера по хранению данных окончательно вышли за рамки простых пар «ключ-значение» в localStorage. Современная мета-ОС требует полноценных баз данных, транзакционной целостности и прямого доступа к файловой системе, чтобы веб-приложения могли конкурировать с нативным софтом по производительности и функциональности.

Браузеры 2026 года предоставляют четыре основных слоя хранения данных, каждый из которых решает свои задачи:

1. **Web Storage (localStorage, sessionStorage)** — для простых строковых данных до 5–10 МБ. Устаревает как основное хранилище.
2. **IndexedDB 3.0** — для структурированных данных, индексов и больших объемов (гигабайты).
3. **OPFS (Origin Private File System)** — для высокопроизводительной работы с файлами, включая запуск SQLite.
4. **File System Access API** — для работы с файлами и папками в нативной файловой системе пользователя.

Эта глава детально разбирает каждый из этих слоев, их применение, производительность, квоты и механизмы управления хранилищем.

---

## 1. IndexedDB 3.0: основное хранилище структурированных данных

**IndexedDB** — это низкоуровневое асинхронное хранилище типа «ключ-значение» с поддержкой индексов, транзакций и бинарных данных (Blob, ArrayBuffer, File). В 2026 году IndexedDB достиг версии 3.0, которая принесла значительные улучшения производительности, новые методы работы с курсорами и улучшенную интеграцию с другими системными API.

### 1.1. Ключевые возможности IndexedDB 3.0

- **Хранение структурированных данных.** IndexedDB может хранить любые данные, поддерживаемые Structured Clone Algorithm: объекты, массивы, даты, регулярные выражения, бинарные данные (ArrayBuffer, Blob), Map, Set.
- **Индексы для быстрого поиска.** Разработчик может создавать индексы по любым полям объектов, что позволяет выполнять быстрые запросы без сканирования всей базы.
- **Транзакции (ACID).** IndexedDB поддерживает транзакции с атомарностью, согласованностью, изоляцией и долговечностью (ACID). Это критически важно для приложений с конкурентным доступом (несколько вкладок/воркеров).
- **Асинхронный API.** Все операции IndexedDB асинхронны и не блокируют основной поток. В 2026 году IndexedDB полностью поддерживает `async/await` через Promise-обертки, что упрощает работу.

### 1.2. Базовое использование IndexedDB 3.0

```javascript
// Открытие базы данных
const dbPromise = indexedDB.open('MyAppDB', 3); // версия 3

dbPromise.onupgradeneeded = (event) => {
    const db = event.target.result;
    
    // Создание хранилища объектов с автогенерируемым ключом
    if (!db.objectStoreNames.contains('users')) {
        const store = db.createObjectStore('users', { keyPath: 'id', autoIncrement: true });
        // Создание индекса для быстрого поиска по email
        store.createIndex('email', 'email', { unique: true });
        store.createIndex('age', 'age');
    }
    
    // Создание хранилища для файлов с бинарными данными
    if (!db.objectStoreNames.contains('files')) {
        db.createObjectStore('files', { keyPath: 'id', autoIncrement: true });
    }
};

// Запись данных
async function saveUser(user) {
    const db = await dbPromise;
    const tx = db.transaction('users', 'readwrite');
    const store = tx.objectStore('users');
    const request = store.add(user);
    await new Promise((resolve, reject) => {
        request.onsuccess = resolve;
        request.onerror = reject;
    });
}

// Поиск по индексу
async function findUserByEmail(email) {
    const db = await dbPromise;
    const tx = db.transaction('users', 'readonly');
    const store = tx.objectStore('users');
    const index = store.index('email');
    const request = index.get(email);
    return new Promise((resolve, reject) => {
        request.onsuccess = () => resolve(request.result);
        request.onerror = reject;
    });
}

// Использование с async/await
const user = await findUserByEmail('john@example.com');
```

### 1.3. IndexedDB 3.0: новые возможности

- **Итераторы и курсоры с async-итераторами.** IndexedDB 3.0 поддерживает асинхронные итераторы (`for await...of`) для курсоров, что упрощает работу с большими наборами данных:

```javascript
const tx = db.transaction('users', 'readonly');
const store = tx.objectStore('users');
const cursor = store.openCursor();

for await (const cursor of cursor) {
    console.log(cursor.value);
}
```

- **Сжатие данных.** Некоторые браузеры (например, Chromium) автоматически сжимают данные в IndexedDB (через алгоритмы Snappy или Zstd) для экономии дискового пространства.
- **Параллельные транзакции.** IndexedDB 3.0 улучшил поддержку параллельных транзакций из разных вкладок и воркеров, снижая блокировки и повышая пропускную способность.

### 1.4. Интеграция с Web Workers

IndexedDB может использоваться из Web Workers, что позволяет выполнять операции с базой данных в фоновом потоке, не нагружая основной поток.

```javascript
// В воркере
const db = await indexedDB.open('MyAppDB', 3);
// Все операции выполняются асинхронно в воркере
```

---

## 2. OPFS (Origin Private File System) и SQLite в браузере

**OPFS (Origin Private File System)** — это выделенная файловая система для каждого origin (домена), доступная через File System Access API. OPFS предоставляет прямой, высокопроизводительный доступ к файлам, которые хранятся в изолированном контейнере на диске пользователя.

### 2.1. Что такое OPFS

OPFS — это виртуальная файловая система, привязанная к origin (домену + протоколу + порту). Файлы в OPFS:

- Хранятся в специальной директории браузера, недоступной для других приложений или пользователя через проводник ОС.
- Имеют высокую производительность чтения/записи, сравнимую с нативными приложениями (в отличие от обычных IndexedDB).
- Доступны только через File System Access API и не видны через обычный File API.

### 2.2. Доступ к OPFS

```javascript
// Получение корневой директории OPFS для текущего origin
const root = await navigator.storage.getDirectory();

// Создание файла
const fileHandle = await root.getFileHandle('data.bin', { create: true });
const writable = await fileHandle.createWritable();
await writable.write(new Uint8Array([1, 2, 3, 4, 5]));
await writable.close();

// Чтение файла
const file = await fileHandle.getFile();
const buffer = await file.arrayBuffer();
```

### 2.3. SQLite поверх OPFS: полноценная реляционная база в браузере

Благодаря OPFS разработчики получили возможность запускать полноценную **SQLite** в браузере через WebAssembly. SQLite — это легковесная реляционная СУБД (без сервера), которая теперь работает с производительностью, близкой к нативной.

**Как это работает:**

1. SQLite скомпилирован в WebAssembly и загружается в браузер.
2. Вместо использования виртуальной файловой системы VFS (которая была медленной), SQLite использует **OPFS VFS** (Origin-Private File System Virtual File System) — прямую обертку над OPFS API.
3. Все операции с базой данных (SELECT, INSERT, UPDATE, DELETE, CREATE TABLE) выполняются локально, без сети, с транзакционной целостностью.

**Библиотеки для SQLite в браузере (2026):**

- **sql.js** — классическая реализация SQLite через Emscripten (WASM).
- **@sqlite.org/sqlite-wasm** — официальная сборка SQLite для браузера от команды SQLite.
- **wa-sqlite** — современная реализация с поддержкой OPFS и многопоточности.

**Пример использования SQLite с OPFS:**

```javascript
import sqlite3 from '@sqlite.org/sqlite-wasm';

// Инициализация SQLite с OPFS VFS
const sqlite = await sqlite3.initialize({
    vfs: 'opfs', // Используем OPFS для хранения
});

const db = new sqlite3.oo1.DB('/myapp.db', 'c');

// Создание таблицы
db.exec(`CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY,
    name TEXT,
    email TEXT UNIQUE,
    age INTEGER
)`);

// Вставка данных
db.exec(`INSERT INTO users (name, email, age) VALUES (?, ?, ?)`, ['John', 'john@example.com', 30]);

// Запрос данных
const rows = db.exec(`SELECT * FROM users WHERE age > ?`, [20]);
console.log(rows);
```

**Преимущества SQLite поверх OPFS:**

- **Производительность.** OPFS обеспечивает скорость записи/чтения, сопоставимую с нативными приложениями, без накладных расходов IndexedDB.
- **Транзакционная целостность.** SQLite гарантирует ACID-транзакции, что критически важно для бизнес-логики.
- **Сложные запросы.** Поддержка JOIN, GROUP BY, подзапросов, оконных функций.
- **Офлайн-режим.** Все данные хранятся локально, приложение работает без интернета.

**Где применяется SQLite в браузере в 2026 году:**

- Локальный кеш для сложных данных (например, контакты, сообщения).
- Офлайн-приложения для управления проектами, заметками, финансами.
- Кеш ответов API с индексацией для быстрого поиска.
- Аналитический движок для обработки больших логов локально.

### 2.4. Производительность OPFS

Измерения 2026 года показывают:

| Операция | IndexedDB | OPFS | Разница |
|----------|-----------|------|---------|
| Запись 1 МБ данных | ~50 мс | ~15 мс | OPFS в 3 раза быстрее |
| Чтение 1 МБ данных | ~30 мс | ~10 мс | OPFS в 3 раза быстрее |
| Транзакция (1000 записей) | ~800 мс | ~200 мс | OPFS в 4 раза быстрее |

OPFS минимизирует накладные расходы на сериализацию и десериализацию, работая напрямую с бинарными данными, что делает его идеальным выбором для высокопроизводительных приложений (видеоредакторы, CAD-системы, базы данных).

---

## 3. File System Access API: работа с нативной файловой системой

**File System Access API** (ранее известный как Native File System API) превращает браузер в полноценную рабочую среду, позволяя веб-приложениям взаимодействовать с файлами и папками пользователя так же, как это делают классические программы.

### 3.1. Открытие файлов и папок

```javascript
// Открытие одного файла через системный диалог
const [fileHandle] = await window.showOpenFilePicker({
    types: [{
        description: 'Text Files',
        accept: { 'text/plain': ['.txt', '.md'] }
    }],
    multiple: false
});

// Чтение содержимого
const file = await fileHandle.getFile();
const content = await file.text();

// Открытие нескольких файлов
const fileHandles = await window.showOpenFilePicker({
    multiple: true
});

// Открытие папки
const folderHandle = await window.showDirectoryPicker();
```

### 3.2. Чтение и запись файлов

```javascript
// Чтение файла как бинарных данных
const file = await fileHandle.getFile();
const buffer = await file.arrayBuffer();

// Запись в файл (требуется разрешение на запись)
const writable = await fileHandle.createWritable({ keepExistingData: true });
await writable.write('Новый текст');
await writable.close();

// Перезапись файла
const writable = await fileHandle.createWritable({ keepExistingData: false });
await writable.write(new Uint8Array([1, 2, 3]));
await writable.close();
```

### 3.3. Работа с папками

```javascript
// Получение доступа к папке
const folderHandle = await window.showDirectoryPicker();

// Чтение содержимого папки
const entries = [];
for await (const entry of folderHandle.values()) {
    entries.push(entry);
}

// Создание нового файла в папке
const newFileHandle = await folderHandle.getFileHandle('new-file.txt', { create: true });
const writable = await newFileHandle.createWritable();
await writable.write('Привет, мир!');
await writable.close();

// Создание подпапки
const subFolderHandle = await folderHandle.getDirectoryHandle('subfolder', { create: true });
```

### 3.4. Сохранение файлов (Save As)

```javascript
// Открытие диалога сохранения
const fileHandle = await window.showSaveFilePicker({
    suggestedName: 'document.txt',
    types: [{
        description: 'Text Files',
        accept: { 'text/plain': ['.txt'] }
    }]
});

const writable = await fileHandle.createWritable();
await writable.write('Содержимое документа');
await writable.close();
```

### 3.5. Права доступа и разрешения

File System Access API использует модель разрешений на основе пользовательского взаимодействия (user gesture). Каждый раз, когда приложение запрашивает доступ к файлу или папке, пользователь должен явно подтвердить это через системный диалог.

```javascript
// Проверка прав на запись
const permission = await fileHandle.requestPermission({
    mode: 'readwrite'
});

if (permission === 'granted') {
    // Разрешение получено
} else {
    // Пользователь отклонил запрос
}
```

**Особенности безопасности:**

- Приложение не может получить доступ к файлам без явного согласия пользователя.
- Доступ ограничен конкретным файлом или папкой — нет доступа ко всей файловой системе.
- Права могут быть отозваны в любое время через настройки браузера.
- Изоляция между разными сайтами (origin) — данные одного сайта не видны другому.

### 3.6. Применение File System Access API в 2026 году

- **Редакторы кода (VSCode в браузере).** Открытие целых проектов, работа с файлами, автосохранение.
- **Графические редакторы (Figma, Photopea).** Открытие и сохранение проектов в нативную файловую систему.
- **CAD-системы (AutoCAD Web, Onshape).** Работа с большими чертежами и проектами.
- **Офисные пакеты (Google Docs, Microsoft 365 Web).** Открытие и сохранение документов на диске.
- **IDE и отладчики.** Открытие логов, конфигурационных файлов.

---

## 4. Хранилище и квоты: управление дисковым пространством

Поскольку браузер теперь может хранить терабайты данных через IndexedDB, OPFS и другие системные вызовы, управление квотами стало важнейшей задачей мета-ОС.

### 4.1. Динамические квоты

Браузеры 2026 года выделяют место на диске **динамически**, основываясь на доступном пространстве и частоте использования приложения.

```javascript
// Получение информации о доступном пространстве
const storageEstimate = await navigator.storage.estimate();
console.log(`Всего: ${storageEstimate.quota / 1024 / 1024 / 1024} GB`);
console.log(`Использовано: ${storageEstimate.usage / 1024 / 1024 / 1024} GB`);
```

- **Квота по умолчанию.** Обычно составляет ~50% от свободного места на диске.
- **Динамическое увеличение.** Если приложение активно используется и хранит данные, браузер может увеличить квоту.
- **Очистка при нехватке места.** Если системе не хватает места, браузер автоматически удаляет наименее используемые данные (сначала из IndexedDB, потом из OPFS).

### 4.2. Persistent Storage (постоянное хранилище)

Веб-приложения могут запрашивать **Persistent Storage** (постоянное хранилище), чтобы их данные не были автоматически удалены браузером при нехватке места.

```javascript
// Запрос постоянного хранилища
const isPersisted = await navigator.storage.persisted();
if (!isPersisted) {
    const result = await navigator.storage.persist();
    if (result) {
        console.log('Хранилище помечено как постоянное');
    }
}
```

**Когда применять Persistent Storage:**

- Офлайн-приложения с критическими данными (заметки, контакты, проекты).
- Приложения, хранящие пользовательские файлы и документы.
- Базы данных SQLite, работающие локально.

**Важно:** запрос постоянного хранилища требует пользовательского жеста (клик, тап) и может быть отклонен браузером, если он считает, что приложение злоупотребляет этим.

### 4.3. Изоляция хранилища (Storage Partitioning)

Чтобы защитить приватность пользователя, браузер изолирует хранилище для каждого сайта (origin). Это предотвращает возможность отслеживания пользователя через общие файлы или куки.

**Storage Partitioning** в 2026 году означает, что:

- **IndexedDB** — изолирована по origin.
- **OPFS** — изолирована по origin.
- **localStorage / sessionStorage** — изолированы по origin.
- **CacheStorage** (используемый Service Workers) — изолирован по origin.
- **Cookies** — изолированы через CHIPS (Cookies Having Independent Partitioned State) и схему Partitioned Cookies.

**Пример:** два сайта `example.com` и `another.com` не могут обратиться к хранилищу друг друга, даже если они загружены в одной вкладке через iframe.

### 4.4. Управление хранилищем пользователем

В настройках современных браузеров (Chromium, Firefox, Safari) пользователь может:

- Просматривать список всех сайтов, хранящих данные.
- Видеть, сколько места занимает каждый сайт (origin).
- Очищать данные отдельных сайтов или всех сразу.
- Управлять квотой для каждого сайта.

**Как это выглядит в браузере 2026 года:**

```
Настройки → Конфиденциальность и безопасность → Управление хранилищем

[example.com]   ─ 2.3 GB (очистить)
[another.org]   ─ 850 MB (очистить)
[myapp.io]      ─ 4.1 GB (очистить)
```

Это делает браузер полностью прозрачным для пользователя в вопросах управления данными, подобно тому как это реализовано в мобильных операционных системах (iOS, Android).

### 4.5. Памятка по выбору хранилища в 2026 году

| Хранилище | Размер | Производительность | Особенности |
|-----------|--------|-------------------|-------------|
| **localStorage** | ≤ 5–10 МБ | Медленная (синхронная) | Только строки. Не использовать для больших данных. |
| **sessionStorage** | ≤ 5–10 МБ | Медленная (синхронная) | Данные живут до закрытия вкладки. |
| **IndexedDB 3.0** | до 50% диска | Средняя (асинхронная) | Структурированные данные, индексы, транзакции. |
| **OPFS** | до 50% диска | **Высокая** (прямой доступ) | Бинарные файлы, SQLite, высокопроизводительные операции. |
| **CacheStorage** | до 50% диска | Средняя | Для кеширования ресурсов (Service Workers). |
| **File System Access** | Вся файловая система | Нативная скорость | Доступ к пользовательским файлам и папкам. |

---

## Итог

В 2026 году хранение данных в браузере вышло далеко за рамки localStorage. IndexedDB 3.0 предоставляет мощное хранилище для структурированных данных с поддержкой индексов и транзакций. OPFS (Origin Private File System) дает прямой высокопроизводительный доступ к файлам, позволяя запускать SQLite в браузере с производительностью, близкой к нативной.

File System Access API открывает доступ к нативной файловой системе пользователя, делая возможными полноценные редакторы, IDE и CAD-системы в браузере. Управление квотами и постоянное хранилище (Persistent Storage) дают разработчикам контроль над дисковым пространством, а изоляция хранилища защищает приватность пользователя.

В следующей главе мы перейдем к стримингу и обработке больших данных через Streams API и Backpressure-механизмы.

---

*Перейти к следующей главе: [Глава 5. Стриминг и обработка больших данных](./chapter-05.md)*