# Глава 11. Navigation API и SPA-маршрутизация без боли

В январе 2026 года Navigation API достиг статуса **Baseline Newly Available**, что означает его поддержку во всех современных браузерных движках: Chrome, Edge, Firefox 147 и Safari 26.2 . Это событие ознаменовало конец эры, когда разработчикам SPA приходилось полагаться на устаревший History API, который Ian Hickson, бывший редактор HTML-спецификации, однажды назвал своей «любимой ошибкой» .

Эта глава детально разбирает Navigation API: его устройство, механизмы перехвата навигации с возможностью префетчинга и предзагрузки, а также встроенные механизмы управления прокруткой и состоянием.

---

## 1. Замена старого History API на декларативную навигацию

History API долгое время служил основным механизмом для маршрутизации в SPA, но имел фундаментальные недостатки, которые делали его использование хрупким и неудобным :

- **Неполный перехват навигации.** History API не позволял обнаружить все типы навигационных событий. Разработчикам приходилось отдельно обрабатывать клики по ссылкам, события `popstate` и программные вызовы `pushState`/`replaceState`.
- **Невозможность инспектировать историю.** History API не предоставлял способа просмотреть весь стек истории — только текущую запись.
- **Непоследовательное поведение `popstate`.** Событие `popstate` не срабатывало при вызовах `pushState` и `replaceState`, что создавало путаницу.
- **Отсутствие управления скроллом.** History API не предоставлял встроенных механизмов для восстановления позиции прокрутки.

Navigation API был спроектирован как полная замена, а не инкрементальное улучшение .

### 1.1. Основные объекты Navigation API

API доступен через глобальное свойство `window.navigation` .

- **`Navigation`** — главный объект, предоставляющий методы для навигации (`navigate()`, `back()`, `forward()`, `traverseTo()`, `reload()`) и свойства для работы с историей (`entries()`, `currentEntry`) .
- **`NavigationHistoryEntry`** — объект, представляющий отдельную запись в истории навигации. Содержит свойства: `key` (уникальный идентификатор записи), `url`, `index` (позиция в стеке), а также метод `getState()` для получения сохраненного состояния .
- **`NavigateEvent`** — событие, которое срабатывает при любой навигации (клик по ссылке, отправка формы, кнопки «Назад»/«Вперед», программный вызов). Обеспечивает единую точку управления всей навигацией .

### 1.2. Инспектирование истории

Navigation API позволяет просматривать полный список записей истории для текущего origin. В отличие от History API, этот список содержит только записи, принадлежащие текущему origin, и не включает навигации внутри iframe или кросс-оригинные переходы .

```javascript
// Получение всех записей истории для текущего origin
const entries = navigation.entries();

// Вывод списка с визуальным отличием текущей записи
entries.forEach(entry => {
    const isCurrent = entry.key === navigation.currentEntry.key;
    console.log(`${isCurrent ? '▶' : '  '} ${entry.url} [${entry.key}]`);
});

// Получение состояния конкретной записи
const state = entries[0].getState();
console.log('Сохраненное состояние первой записи:', state);
```

### 1.3. Программная навигация

Все методы навигации возвращают объект с двумя промисами: `{ committed, finished }` .

- **`committed`** — выполняется, когда URL изменился и создана новая запись истории.
- **`finished`** — выполняется, когда все обработчики `intercept()` завершили свою работу.

```javascript
// Навигация на новый URL
const result = navigation.navigate('/products/123');

await result.committed;
console.log('URL изменен, запись создана');

await result.finished;
console.log('Все обработчики перехвата завершены');

// Возврат на предыдущую страницу
navigation.back();

// Переход к конкретной записи по ключу
const targetKey = navigation.entries()[0].key;
navigation.traverseTo(targetKey);

// Перезагрузка текущей страницы
navigation.reload();
```

### 1.4. Событие navigate: единая точка управления

Главное отличие Navigation API — единое событие `navigate`, которое срабатывает для всех типов навигаций: клики по ссылкам, отправка форм, кнопки «Назад»/«Вперед», программные вызовы .

```javascript
navigation.addEventListener('navigate', (event) => {
    console.log('Тип навигации:', event.navigationType); // 'push', 'replace', 'traverse', 'reload'
    console.log('Целевой URL:', event.destination.url);
    console.log('Инициировано пользователем:', event.userInitiated);
    console.log('Это переход по hash-части:', event.hashChange);
    console.log('Можно перехватить:', event.canIntercept);
});
```

### 1.5. Готовность к использованию в SPA-роутерах

На момент 2026 года популярные SPA-роутеры (React Router, TanStack Router) ведут обсуждения о переходе на Navigation API в качестве бэкенда для маршрутизации . Однако API находится на более низком уровне, чем эти фреймворки, предоставляя платформенные примитивы, на которых могут строиться высокоуровневые абстракции .

---

## 2. Intercepting навигации: префетчинг и предзагрузка ресурсов

Ключевая возможность Navigation API — метод `intercept()`, который позволяет перехватывать навигацию и полностью контролировать ее выполнение . Это заменяет громоздкие и ненадежные подходы с ручным перехватом кликов и управлением историей.

### 2.1. Метод intercept(): базовое использование

Метод `intercept()` принимает объект с двумя основными обработчиками :

- **`precommitHandler`** — выполняется до изменения URL. Идеален для загрузки данных, при этом старый контент остается видимым.
- **`handler`** — выполняется после изменения URL. Используется для переключения контента.

```javascript
navigation.addEventListener('navigate', (event) => {
    // Проверяем, можно ли перехватить навигацию
    if (!event.canIntercept) return;

    // Перехватываем навигацию
    event.intercept({
        // precommitHandler: загрузка данных до смены URL
        precommitHandler: async () => {
            // Старый контент все еще виден
            await loadDataForPage(event.destination.url);
        },
        // handler: смена контента после смены URL
        handler: async () => {
            // URL уже изменился, заменяем контент
            await renderNewPage(event.destination.url);
        }
    });
});
```

### 2.2. Управление состоянием и историей

В обработчике можно изменять тип навигации (`push` или `replace`) и сохранять состояние :

```javascript
navigation.addEventListener('navigate', (event) => {
    if (!event.canIntercept) return;

    event.intercept({
        precommitHandler: (controller) => {
            // Изменение типа навигации на 'replace'
            controller.redirect(event.destination.url, {
                history: 'replace',
                state: { pageVisited: true, timestamp: Date.now() },
                info: { from: 'navigation-interceptor' }
            });
        },
        handler: async () => {
            // Рендеринг страницы
            await renderPage(event.destination.url);
        }
    });
});
```

**Важное ограничение:** `precommitHandler` в Safari 26.2 пока не поддерживается .

### 2.3. Обработка ошибок навигации

Navigation API предоставляет события `navigatesuccess` и `navigateerror` для централизованной обработки успешных и неудачных навигаций :

```javascript
navigation.addEventListener('navigatesuccess', (event) => {
    console.log('Навигация успешно завершена');
    // Очистка состояния загрузки, закрытие индикаторов
});

navigation.addEventListener('navigateerror', (event) => {
    console.error('Ошибка навигации:', event.error);
    // Показ сообщения об ошибке пользователю
    showErrorMessage('Не удалось загрузить страницу');
});
```

### 2.4. Отмена навигации

Для отмены навигации используется `event.preventDefault()` :

```javascript
navigation.addEventListener('navigate', (event) => {
    if (!event.canIntercept) return;

    // Проверка: пользователь авторизован?
    if (!isUserLoggedIn() && event.destination.url.includes('/dashboard')) {
        event.preventDefault(); // Отмена навигации
        redirectToLogin();
        return;
    }

    event.intercept({
        handler: async () => {
            await renderPage(event.destination.url);
        }
    });
});
```

### 2.5. Префетчинг и предзагрузка ресурсов

Navigation API не предоставляет встроенных методов префетчинга, но его архитектура отлично сочетается со стандартными механизмами предварительной загрузки:

**Speculation Rules API.** Используется для префетчинга и предварительного рендеринга страниц, на которые пользователь может перейти .

```html
<script type="speculationrules">
{
  "prefetch": [
    {
      "source": "list",
      "urls": ["/products", "/about", "/contact"],
      "requires": ["anonymous-client-ip-when-cross-origin"]
    }
  ],
  "prerender": [
    {
      "source": "list",
      "urls": ["/dashboard"],
      "eagerness": "moderate"
    }
  ]
}
</script>
```

**Интеграция с `precommitHandler`.** В обработчике можно выполнять загрузку данных для страницы назначения, пока старая страница все еще видна:

```javascript
navigation.addEventListener('navigate', (event) => {
    if (!event.canIntercept) return;

    event.intercept({
        precommitHandler: async () => {
            // Загрузка данных для страницы назначения
            // Старый контент ВСЕ ЕЩЕ виден пользователю
            const data = await fetchPageData(event.destination.url);
            // Сохраняем данные для handler
            window._prefetchedData = data;
        },
        handler: async () => {
            // Используем предзагруженные данные
            renderPage(window._prefetchedData);
        }
    });
});
```

---

## 3. Scroll Restoration и управление состояниями переходов

Одной из самых сложных задач при работе с History API было управление прокруткой. Navigation API решает эту проблему встроенными механизмами, а также предоставляет интеграцию с View Transitions API для бесшовных анимаций переходов между страницами.

### 3.1. Управление прокруткой

Метод `intercept()` принимает параметры `scroll` и `focusReset` для управления поведением браузера после навигации :

```javascript
navigation.addEventListener('navigate', (event) => {
    if (!event.canIntercept) return;

    event.intercept({
        scroll: 'manual',    // 'after-transition' (по умолчанию) или 'manual'
        focusReset: 'manual', // 'after-transition' (по умолчанию) или 'manual'
        handler: async () => {
            await renderPage(event.destination.url);

            // Ручное управление прокруткой
            if (event.destination.url.includes('#section')) {
                const targetId = event.destination.url.split('#')[1];
                const element = document.getElementById(targetId);
                if (element) {
                    element.scrollIntoView({ behavior: 'smooth' });
                }
            } else {
                window.scrollTo({ top: 0, behavior: 'smooth' });
            }
        }
    });
});
```

**Ручной вызов прокрутки.** Если установлен `scroll: 'manual'`, разработчик может вызвать `event.scroll()` для запуска стандартной логики восстановления прокрутки в удобный момент :

```javascript
navigation.addEventListener('navigate', (event) => {
    if (!event.canIntercept) return;

    event.intercept({
        scroll: 'manual',
        handler: async () => {
            await renderPage(event.destination.url);

            // Анимация завершена — восстанавливаем прокрутку
            setTimeout(() => {
                event.scroll(); // Восстанавливает позицию прокрутки браузера
            }, 300);
        }
    });
});
```

### 3.2. Состояния записей истории

Каждая запись истории может хранить произвольное состояние, которое восстанавливается при возврате на страницу :

```javascript
// Сохранение состояния при навигации
navigation.navigate('/settings', {
    state: {
        activeTab: 'profile',
        scrollPosition: window.scrollY,
        expandedSections: ['general', 'notifications']
    }
});

// Восстановление состояния при возврате
navigation.addEventListener('navigate', (event) => {
    if (!event.canIntercept) return;

    const targetEntry = navigation.entries().find(
        entry => entry.key === event.destination.key
    );

    if (targetEntry) {
        const state = targetEntry.getState();
        if (state) {
            // Восстановление состояния
            restoreUIState(state);
        }
    }
});
```

### 3.3. Интеграция с View Transitions API

Navigation API и View Transitions API вместе обеспечивают бесшовные анимации переходов между страницами:

```javascript
navigation.addEventListener('navigate', (event) => {
    if (!event.canIntercept) return;

    event.intercept({
        handler: async () => {
            // Запуск перехода между состояниями
            const transition = document.startViewTransition(() => {
                renderPage(event.destination.url);
            });
            
            await transition.finished;
        }
    });
});
```

**Делегирование восстановления прокрутки.** View Transitions API может откладывать восстановление состояния (включая позицию прокрутки) до захвата старого состояния, что обеспечивает плавные переходы без рывков:

```css
/* CSS для View Transitions */
::view-transition-old(root) {
    animation: fade-out 0.3s ease-in;
}

::view-transition-new(root) {
    animation: fade-in 0.3s ease-out;
}

@keyframes fade-out {
    from { opacity: 1; }
    to { opacity: 0; }
}

@keyframes fade-in {
    from { opacity: 0; transform: scale(0.98); }
    to { opacity: 1; transform: scale(1); }
}
```

---

## Итог

Navigation API в 2026 году представляет собой полноценную замену устаревшему History API, обеспечивая разработчиков инструментами для создания надежной, производительной маршрутизации в SPA:

- **Централизованное управление навигацией.** Событие `navigate` обрабатывает все типы навигаций (ссылки, формы, кнопки «Назад»/«Вперед», программные вызовы) в одном месте, заменяя фрагментированный подход с клик-листенерами и `popstate` .
- **Перехват навигации.** Метод `intercept()` с `precommitHandler` (загрузка данных до смены URL) и `handler` (смена контента) обеспечивает полный контроль над навигацией.
- **Управление состоянием и прокруткой.** Встроенные механизмы `scroll` и `focusReset`, а также возможность хранения состояния в записях истории упрощают сложные сценарии с восстановлением UI.
- **Интеграция с View Transitions.** Анимации переходов между состояниями становятся простыми благодаря интеграции с View Transitions API.

В следующей главе мы перейдем к вопросам безопасности и приватности, рассмотрев современные модели изоляции и управление разрешениями.

---

*Перейти к следующей главе: [Глава 12. Модель изоляции и Permissions API](./chapter-12.md)*