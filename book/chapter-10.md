# Глава 10. Service Workers как прокси-сервер

Service Worker — это специализированный тип Web Worker, который действует как программируемый прокси-сервер между браузером и сетью. Он перехватывает сетевые запросы, кеширует ресурсы, обеспечивает работу приложения в офлайн-режиме и обрабатывает push-уведомления. В 2026 году Service Workers стали зрелой технологией, поддерживаемой всеми основными браузерами, включая поддержку JavaScript-модулей для лучшей организации кода .

Эта глава последовательно разбирает три ключевых аспекта Service Workers: стратегии кеширования (с акцентом на Stale-While-Revalidate и Background Sync), превращение веб-приложений в полноценные приложения ОС через PWA и работу с уведомлениями через Push API.

---

## 1. Стратегии кеширования нового поколения

Service Worker предоставляет разработчику полный контроль над кешированием ресурсов. Выбор правильной стратегии кеширования определяет скорость загрузки, офлайн-доступность и согласованность данных. В 2026 году библиотеки и инструменты предлагают три основные стратегии, каждая из которых решает свой класс задач .

### 1.1. Cache-First: скорость в ущерб свежести

**Принцип работы:** при запросе браузер сначала проверяет кеш. Если ресурс найден и не истек, он возвращается из кеша. Если ресурс отсутствует или устарел, выполняется сетевой запрос, и ответ сохраняется в кеш .

**Когда применять:** статические ресурсы, которые редко меняются: CSS-файлы, JavaScript-бандлы, шрифты, изображения-шаблоны, иконки приложения .

**Пример реализации:**

```javascript
// Cache-First стратегия для статических ассетов
async function cacheFirst(request) {
    const cache = await caches.open('static-v1');
    const cachedResponse = await cache.match(request);
    
    if (cachedResponse) {
        return cachedResponse;
    }
    
    try {
        const networkResponse = await fetch(request);
        cache.put(request, networkResponse.clone());
        return networkResponse;
    } catch (error) {
        // Офлайн-режим — возвращаем fallback страницу
        return caches.match('/offline.html');
    }
}
```

### 1.2. Network-First: свежесть в ущерб скорости

**Принцип работы:** при запросе браузер сначала пытается получить ресурс из сети. Если сеть недоступна, используется кеш как fallback .

**Когда применять:** API-эндпоинты с критически важными данными, которые должны быть актуальными (например, баланс пользователя, состояние корзины).

### 1.3. Stale-While-Revalidate: баланс скорости и свежести

**Stale-While-Revalidate** (SWR) — стратегия, которая обеспечивает мгновенный ответ из кеша при одновременном фоновом обновлении данных из сети. Это лучший баланс между производительностью и актуальностью данных для динамического контента .

**Принцип работы:**

1. При запросе браузер немедленно возвращает кешированную версию (даже если она устарела).
2. Асинхронно отправляется сетевой запрос для получения свежей версии.
3. Когда свежая версия получена, она обновляется в кеше для будущих запросов.

**Алгоритм с TTL (Time-To-Live):**

Более точная реализация SWR в 2026 году использует два порога устаревания :

- **Fresh TTL (обычно 5–10 минут):** ресурс считается свежим, возвращается из кеша без фонового обновления.
- **Stale TTL (обычно 1 час):** ресурс считается устаревшим, но пригодным к использованию. Возвращается из кеша, но одновременно выполняется фоновое обновление.
- **За пределами Stale TTL:** ресурс считается слишком устаревшим, требуется принудительный сетевой запрос.

**Пример реализации SWR с TTL:**

```javascript
// Stale-While-Revalidate с TTL-логикой
async function staleWhileRevalidate(request, freshTTL = 300, staleTTL = 3600) {
    const cache = await caches.open('dynamic-v1');
    const cachedResponse = await cache.match(request);
    
    if (cachedResponse) {
        const cachedTime = cachedResponse.headers.get('sw-cache-time');
        const age = Date.now() - parseInt(cachedTime);
        
        // Свежий — возвращаем без обновления
        if (age < freshTTL * 1000) {
            return cachedResponse;
        }
        
        // Устаревший, но пригодный к использованию — возвращаем и обновляем фоном
        if (age < staleTTL * 1000) {
            updateCacheInBackground(request);
            return cachedResponse;
        }
        // Слишком старый — не используем кеш
    }
    
    // Нет кеша или слишком старый — сетевой запрос
    const networkResponse = await fetch(request);
    const clonedResponse = networkResponse.clone();
    
    const headers = new Headers(clonedResponse.headers);
    headers.set('sw-cache-time', Date.now());
    const responseWithTime = new Response(clonedResponse.body, {
        status: clonedResponse.status,
        statusText: clonedResponse.statusText,
        headers
    });
    cache.put(request, responseWithTime);
    return networkResponse;
}

async function updateCacheInBackground(request) {
    try {
        const cache = await caches.open('dynamic-v1');
        const response = await fetch(request);
        const clonedResponse = response.clone();
        const headers = new Headers(clonedResponse.headers);
        headers.set('sw-cache-time', Date.now());
        const responseWithTime = new Response(clonedResponse.body, {
            status: clonedResponse.status,
            statusText: clonedResponse.statusText,
            headers
        });
        cache.put(request, responseWithTime);
    } catch (error) {
        // Фоновое обновление не удалось — кеш остается без изменений
    }
}
```

### 1.4. Background Sync: синхронизация при восстановлении сети

Background Sync — это механизм, который позволяет Service Worker откладывать выполнение действий до момента, когда устройство снова подключится к интернету. Это критически важно для офлайн-приложений, где пользователь может отправлять данные (формы, комментарии, заказы) без активного соединения .

**Как работает Background Sync:**

1. Приложение регистрирует событие синхронизации в Service Worker с указанием тега.
2. Если запрос не удался из-за отсутствия сети, Service Worker регистрирует задачу синхронизации.
3. Когда устройство восстанавливает соединение, браузер пробуждает Service Worker и вызывает обработчик синхронизации.
4. В обработчике выполняется повторная отправка данных.
5. Если данные отправлены успешно, задача синхронизации завершается. Если нет — браузер будет повторять попытки с экспоненциальной задержкой.

**Пример реализации Background Sync:**

```javascript
// Регистрация синхронизации в основном потоке
navigator.serviceWorker.ready.then(registration => {
    registration.sync.register('sync-orders')
        .then(() => console.log('Синхронизация зарегистрирована'));
});

// Обработчик синхронизации в Service Worker
self.addEventListener('sync', event => {
    if (event.tag === 'sync-orders') {
        event.waitUntil(syncOrders());
    }
});

async function syncOrders() {
    const cache = await caches.open('pending-orders');
    const requests = await cache.keys();
    
    for (const request of requests) {
        try {
            const response = await fetch(request);
            if (response.ok) {
                await cache.delete(request);
            }
        } catch (error) {
            // Запрос не удался — останется в кеше для следующей синхронизации
        }
    }
}
```

### 1.5. Таблица стратегий кеширования

| Стратегия | Скорость | Свежесть | Офлайн | Когда применять |
|-----------|----------|----------|--------|-----------------|
| **Cache-First** | Мгновенная | Низкая | ✅ | Статика (CSS, JS, иконки)  |
| **Network-First** | Зависит от сети | Высокая | ⚠️ Fallback | Критические API  |
| **Stale-While-Revalidate** | Мгновенная | Средняя | ✅ | API, изображения, динамический контент  |
| **Cache-Only** | Мгновенная | Фиксированная | ✅ | App Shell, офлайн-шаблоны  |
| **Network-Only** | Зависит от сети | Максимальная | ❌ | Платежи, чувствительные данные |

---

## 2. Прогрессивные Web-приложения (PWA) как приложения ОС

В 2026 году PWA (Progressive Web Applications) стали полноценной альтернативой нативным приложениям. Поддержка PWA интегрирована во все современные ОС: iOS, Android, Windows, macOS и даже в отечественную ОС "Аврора" .

### 2.1. Требования к PWA

Чтобы веб-приложение стало устанавливаемым PWA, оно должно соответствовать трем основным требованиям:

1. **Web App Manifest.** JSON-файл, описывающий приложение: название, иконки, цвета, режим отображения (fullscreen/standalone).
2. **Service Worker.** Кеширует ресурсы и обеспечивает офлайн-работу.
3. **HTTPS.** Приложение должно обслуживаться через защищенное соединение.

**Пример Web App Manifest (2026):**

```json
{
    "name": "Мое приложение",
    "short_name": "Мое приложение",
    "description": "Пример PWA для главы 10",
    "start_url": "/",
    "display": "standalone",
    "background_color": "#ffffff",
    "theme_color": "#1976d2",
    "orientation": "portrait-primary",
    "icons": [
        {
            "src": "/icons/icon-192.png",
            "sizes": "192x192",
            "type": "image/png",
            "purpose": "any maskable"
        },
        {
            "src": "/icons/icon-512.png",
            "sizes": "512x512",
            "type": "image/png",
            "purpose": "any maskable"
        }
    ],
    "categories": ["productivity", "utilities"],
    "shortcuts": [
        {
            "name": "Новый документ",
            "url": "/new",
            "icons": [{ "src": "/icons/new.png", "sizes": "96x96" }]
        }
    ]
}
```

### 2.2. Установка PWA на разных платформах в 2026 году

| Платформа | Способ установки | Особенности |
|-----------|------------------|-------------|
| **Chrome (Android/Windows)** | Автоматический баннер в адресной строке | Поддержка fullscreen, офлайн, уведомлений |
| **Safari (iOS)** | Кнопка "Share" → "Add to Home Screen" | Уведомления работают только после установки  |
| **Safari (macOS)** | Кнопка "Share" → "Add to Dock" | Уведомления работают без установки  |
| **Edge (Windows)** | Баннер в адресной строке или меню Apps | Интеграция с магазином Microsoft Store |
| **ОС "Аврора"** | Установка из браузера | Полноценная поддержка PWA  |

### 2.3. JavaScript-модули в Service Workers (Baseline 2026)

В январе 2026 года поддержка JavaScript-модулей в Service Workers стала доступна во всех основных браузерных движках и вошла в Baseline . Это позволяет использовать стандартные операторы `import` и `export` внутри Service Worker, что упрощает организацию кода и совместное использование модулей между основным потоком и воркером.

```javascript
// Регистрация Service Worker как модуля
navigator.serviceWorker.register('/sw.js', { type: 'module' });

// sw.js — импорт модулей
import { cacheFirst, staleWhileRevalidate } from './strategies.js';
import { handlePushNotification } from './notifications.js';

self.addEventListener('fetch', event => {
    if (event.request.url.includes('/api/')) {
        event.respondWith(staleWhileRevalidate(event.request));
    } else {
        event.respondWith(cacheFirst(event.request));
    }
});

self.addEventListener('push', event => {
    event.waitUntil(handlePushNotification(event.data));
});
```

### 2.4. PWA-библиотеки 2026 года

В 2026 году выбор библиотек для PWA строится вокруг фреймворков :

| Библиотека | Движок | Для чего |
|------------|--------|----------|
| **Serwist 9** | Современный наследник Workbox | Next.js 14+, App Router  |
| **next-pwa 5.6** | Workbox | Next.js (поддержка только legacy) |
| **vite-plugin-pwa** | Workbox | Vite + React SPA  |
| **Workbox 7** | Google | Нижнеуровневый, фреймворк-агностик |
| **swimple** | Нативный SW | Простые приложения, Tiny-размер  |

**Установка PWA в 2026 году:**

Современные браузеры показывают автоматический баннер установки, если PWA соответствует критериям. Альтернативно, разработчик может использовать событие `beforeinstallprompt` для кастомной кнопки установки:

```javascript
// Кастомная установка через beforeinstallprompt
let deferredPrompt;

window.addEventListener('beforeinstallprompt', event => {
    event.preventDefault();
    deferredPrompt = event;
    document.querySelector('#install-button').style.display = 'block';
});

document.querySelector('#install-button').addEventListener('click', async () => {
    if (deferredPrompt) {
        const result = await deferredPrompt.prompt();
        if (result.outcome === 'accepted') {
            console.log('Приложение установлено');
        }
        deferredPrompt = null;
    }
});
```

---

## 3. Push API и уведомления с высоким приоритетом

Push API позволяет серверу отправлять уведомления пользователю даже при закрытом браузере. В 2026 году эта технология поддерживается всеми основными браузерами, хотя на iOS есть важное ограничение: push-уведомления работают только после установки PWA на домашний экран .

### 3.1. Архитектура Push-уведомлений

Push-уведомления используют push-сервис (браузерный) между сервером приложения и клиентом. Этот сервис управляет доставкой уведомлений, сохраняя состояние подписок и буферизируя сообщения для офлайн-устройств.

**Поток работы:**

1. **Подписка.** Клиент (браузер) запрашивает у пользователя разрешение на уведомления. Если разрешение получено, браузер генерирует объект PushSubscription, который содержит эндпоинт (URL push-сервиса) и ключи шифрования . Подписка отправляется на сервер приложения.

2. **Отправка уведомления.** Сервер приложения использует эндпоинт для отправки уведомления через push-сервис. Данные шифруются с использованием публичного ключа подписки.

3. **Доставка.** Push-сервис доставляет уведомление на устройство клиента. Если устройство офлайн, сообщение сохраняется в очереди и доставляется при восстановлении соединения.

4. **Обработка.** Service Worker клиента получает событие `push`, обрабатывает данные и показывает уведомление через Service Worker Registration.

### 3.2. Подписка на уведомления

```javascript
// Запрос разрешения на уведомления
const permission = await Notification.requestPermission();

if (permission === 'granted') {
    // Регистрация Service Worker
    const registration = await navigator.serviceWorker.ready;
    
    // Подписка на Push
    const subscription = await registration.pushManager.subscribe({
        userVisibleOnly: true,
        applicationServerKey: urlBase64ToUint8Array(publicVapidKey)
    });
    
    // Отправка подписки на сервер
    await fetch('/api/subscribe', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(subscription)
    });
}

// Вспомогательная функция для преобразования ключа
function urlBase64ToUint8Array(base64String) {
    const padding = '='.repeat((4 - base64String.length % 4) % 4);
    const base64 = (base64String + padding).replace(/-/g, '+').replace(/_/g, '/');
    const rawData = window.atob(base64);
    const outputArray = new Uint8Array(rawData.length);
    for (let i = 0; i < rawData.length; ++i) {
        outputArray[i] = rawData.charCodeAt(i);
    }
    return outputArray;
}
```

### 3.3. Обработка Push-уведомлений в Service Worker

Service Worker обрабатывает входящие уведомления через событие `push`. В 2026 году типичная реализация выглядит так:

```javascript
// Обработка push-события в Service Worker
self.addEventListener('push', event => {
    let data = { title: 'Уведомление', body: 'Новое сообщение' };
    
    if (event.data) {
        try {
            data = event.data.json();
        } catch (error) {
            data = { title: 'Новое уведомление', body: event.data.text() };
        }
    }
    
    const options = {
        body: data.body,
        icon: '/icons/icon-192.png',
        badge: '/icons/badge-72.png',
        vibrate: [200, 100, 200],
        data: { url: data.url || '/' },
        actions: data.actions || [
            { action: 'view', title: 'Открыть' },
            { action: 'dismiss', title: 'Закрыть' }
        ],
        requireInteraction: data.urgent || false,
        tag: data.tag || 'default',
        renotify: data.renotify || false
    };
    
    event.waitUntil(
        self.registration.showNotification(data.title, options)
    );
});

// Обработка клика по уведомлению
self.addEventListener('notificationclick', event => {
    event.notification.close();
    
    const url = event.notification.data.url || '/';
    const action = event.action;
    
    if (action === 'view' || action === '') {
        event.waitUntil(
            clients.openWindow(url)
        );
    } else if (action === 'dismiss') {
        // Просто закрываем уведомление
    }
});
```

### 3.4. Приоритеты уведомлений

В 2026 году Push-уведомления поддерживают настройку приоритета (urgency) для управления доставкой :

- **`very-low`:** Фоновые уведомления, доставляются в составе групп, могут задерживаться. Для аналитики и неважных обновлений.
- **`low`:** Низкий приоритет, допустима небольшая задержка.
- **`normal`:** Стандартный приоритет, используется по умолчанию.
- **`high`:** Высокий приоритет, уведомление должно быть доставлено как можно быстрее. Используется для критических событий: платежи, проверка кода, срочные сообщения .

**Установка приоритета при отправке с сервера:**

При отправке уведомления через push-сервис сервер может передать приоритет через стандартные поля Web Push Protocol:

```javascript
// Пример отправки с высоким приоритетом на сервере (Node.js)
import webpush from 'web-push';

webpush.sendNotification(subscription, JSON.stringify({
    title: 'Срочно!',
    body: 'Ваш платеж подтвержден',
    urgent: true,
    actions: [{ action: 'view', title: 'Проверить' }]
}), {
    urgency: 'high', // high | normal | low
    TTL: 86400 // время жизни в секундах (максимум 4 недели)
});
```

---

## Итог

Service Workers в 2026 году — это зрелая технология, которая превращает веб-приложения в полноценные приложения операционной системы:

- **Стратегии кеширования** — Cache-First для статики, Network-First для критических API, Stale-While-Revalidate для баланса скорости и свежести. Background Sync позволяет выполнять фоновые операции при восстановлении сети.
- **PWA как приложения ОС.** Благодаря Web App Manifest и Service Workers приложения устанавливаются на домашний экран (iOS, Android, Windows, macOS, "Аврора") и работают как нативные. В 2026 году Service Workers поддерживают JavaScript-модули, упрощая организацию кода .
- **Push-уведомления.** Позволяют серверу отправлять сообщения даже при закрытом браузере. На iOS уведомления работают только после установки PWA . Приоритет уведомлений (`very-low` до `high`) управляет срочностью доставки .

В следующей главе мы рассмотрим Navigation API — новый стандарт для управления навигацией в SPA, который пришел на смену устаревшему History API.

---

*Перейти к следующей главе: [Глава 11. Navigation API и SPA-маршрутизация без боли](./chapter-11.md)*