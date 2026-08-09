# Глава 5. Стриминг и обработка больших данных

В 2026 году обработка потоковых данных стала одной из ключевых компетенций веб-разработчика. Современные веб-приложения работают с огромными объемами информации: видеопотоки, большие наборы данных, ответы от LLM (Large Language Models). Традиционный подход, при котором все данные сначала загружаются в память целиком, а затем обрабатываются, приводит к высокому потреблению памяти, задержкам и снижению отзывчивости интерфейса.

**Streams API** предоставляет унифицированное решение для работы с потоковыми данными, позволяя обрабатывать информацию частями (чанками) по мере её поступления, без необходимости загружать всё в память целиком. Эта технология стала неотъемлемой частью браузера как мета-ОС и лежит в основе множества современных сценариев — от рендеринга HTML по частям до обработки данных в реальном времени.

---

## 1. Streams API: ReadableStream, WritableStream, TransformStream

Стандарт Streams API определяет три базовых интерфейса для работы с потоковыми данными:

- **ReadableStream** — источник данных, из которого можно читать.
- **WritableStream** — приемник данных, в который можно записывать.
- **TransformStream** — преобразователь, сочетающий в себе ReadableStream и WritableStream, позволяющий модифицировать данные по мере их прохождения.

Базовые операции (`getReader()`, `getWriter()`, `pipeThrough()`, `pipeTo()`) — Baseline и работают одинаково во всех трёх движках. А вот у одной конкретной возможности, разобранной ниже — async-итерации потоков, — статус поддержки иной, и на это стоит обратить особое внимание.

### 1.1. ReadableStream: чтение данных по частям

ReadableStream — это объект, представляющий источник данных, который можно читать асинхронно, по частям (чанкам). Примеры источников: ответы от сервера (`fetch()`), чтение файлов, пользовательские потоки данных.

**Создание ReadableStream с произвольным источником:**

```javascript
const readableStream = new ReadableStream({
  start(controller) {
    // Инициализация источника
    controller.enqueue('Первый чанк');
    controller.enqueue('Второй чанк');
    controller.close();
  },
  pull(controller) {
    // Вызывается, когда читатель запрашивает данные
  },
  cancel(reason) {
    // Отмена потока (читатель потерял интерес)
  }
});
```

**Чтение данных из потока через async-итерацию (современный подход) — и важная оговорка про статус поддержки:**

Здесь стоит быть точным сразу в двух вещах. Во-первых, по спецификации WHATWG Streams протокол async-итератора определён **на самом объекте `ReadableStream`**, а не на `reader`, который возвращает `getReader()`. Поэтому итерировать нужно сам поток напрямую, не блокируя его предварительным вызовом `getReader()`:

```javascript
// Правильно: async-итерация самого ReadableStream, без getReader()
for await (const chunk of readableStream) {
  console.log(chunk);
}
```

Во-вторых, эта возможность **пока не имеет статуса Baseline**. Она поддерживается в Chrome/Edge (начиная с версии 124) и Firefox (начиная с версии 110), но по состоянию на август 2026 года **не реализована ни в одной стабильной версии Safari** — только в Safari Technology Preview. Для кода, который должен одинаково работать во всех браузерах, нужен fallback через классический `reader.read()`:

```javascript
async function readStreamCrossBrowser(stream, onChunk) {
  if (Symbol.asyncIterator in stream) {
    // Chrome, Edge, Firefox — прямая async-итерация
    for await (const chunk of stream) {
      onChunk(chunk);
    }
    return;
  }
  // Fallback для Safari — ручной цикл через reader
  const reader = stream.getReader();
  try {
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      onChunk(value);
    }
  } finally {
    reader.releaseLock();
  }
}
```

**Использование встроенных возможностей:**

```javascript
// Чтение тела HTTP-ответа как поток
const response = await fetch('/api/large-data');
const reader = response.body.getReader();

while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  console.log('Получено байт:', value.length);
}
```

Обратите внимание: пример выше сознательно использует классический цикл `reader.read()`, а не async-итерацию — именно потому, что этот способ работает во всех браузерах уже сегодня, без оглядки на статус Safari.

### 1.2. WritableStream: запись данных по частям

WritableStream — это объект, представляющий приемник, в который можно асинхронно записывать данные по частям.

**Создание WritableStream:**

```javascript
const writableStream = new WritableStream({
  write(chunk) {
    // Обработка чанка
    console.log('Записан чанк:', chunk);
  },
  close() {
    console.log('Поток закрыт');
  },
  abort(reason) {
    console.error('Поток прерван:', reason);
  }
});
```

**Запись в поток:**

```javascript
const writer = writableStream.getWriter();
await writer.write('Чанк 1');
await writer.write('Чанк 2');
await writer.close();
```

### 1.3. TransformStream: преобразование данных

TransformStream — это конвейер, который связывает ReadableStream и WritableStream, применяя преобразование к каждому чанку данных, проходящему через него.

**Создание TransformStream:**

```javascript
const transformStream = new TransformStream({
  transform(chunk, controller) {
    // Преобразование чанка
    const transformed = chunk.toUpperCase();
    controller.enqueue(transformed);
  }
});
```

**Использование TransformStream для преобразования данных:**

```javascript
const uppercaseStream = new TransformStream({
  transform(chunk, controller) {
    controller.enqueue(chunk.toUpperCase());
  }
});

const transformedStream = readableStream.pipeThrough(uppercaseStream);

// Кросс-браузерное чтение результата (см. оговорку про Safari в разделе 1.1)
const reader = transformedStream.getReader();
while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  console.log(value); // данные в верхнем регистре
}
```

### 1.4. Пайплайны: соединение потоков

Главная сила Streams API — возможность соединять потоки в пайплайны (pipe chains), где данные проходят через последовательность преобразований. Методы `pipeThrough()` и `pipeTo()` — Baseline, кросс-браузерны и не имеют отношения к оговорке про async-итерацию из раздела 1.1.

```javascript
// Пайплайн: источник -> преобразование 1 -> преобразование 2 -> приемник
await readableStream
  .pipeThrough(new TransformStream({
    transform(chunk, controller) {
      controller.enqueue(chunk.trim());
    }
  }))
  .pipeThrough(new TransformStream({
    transform(chunk, controller) {
      controller.enqueue(JSON.parse(chunk));
    }
  }))
  .pipeTo(writableStream);
```

**Async-итерация результата пайплайна (Chrome/Edge/Firefox; для Safari — fallback из раздела 1.1):**

```javascript
const pipeline = readableStream
  .pipeThrough(new TransformStream(/* преобразование */))
  .pipeThrough(new TransformStream(/* еще преобразование */));

for await (const chunk of pipeline) {
  // обработка каждого преобразованного чанка
  // в Safari на 2026 год этот синтаксис ещё не работает — используйте readStreamCrossBrowser() из раздела 1.1
}
```

---

## 2. Backpressure (Противодавление): управление потоками данных

Backpressure — это механизм управления потоком данных, который предотвращает переполнение памяти при несоответствии скоростей производителя и потребителя данных. Если производитель генерирует данные быстрее, чем потребитель успевает их обрабатывать, буфер потоков начинает расти, что может привести к исчерпанию памяти (OOM — Out of Memory).

В 2026 году ошибки игнорирования backpressure остаются одной из распространённых причин утечек памяти в потоковых приложениях, особенно в сложных системах с множеством вложенных потоков.

### 2.1. Как работает Backpressure в Web Streams

В стандарте Web Streams backpressure реализуется через механизм **`desiredSize`** — размер свободного места в очереди потока.

- **Положительный `desiredSize`** — в очереди есть место, можно продолжать отправлять данные.
- **`desiredSize <= 0`** — очередь заполнена, производитель должен приостановить отправку.

**Параметр `highWaterMark`** — задает максимальный размер очереди (в чанках или байтах). При превышении `highWaterMark` поток сигнализирует о необходимости снизить скорость.

```javascript
// Настройка стратегии с highWaterMark (количество чанков)
const strategy = new CountQueuingStrategy({ highWaterMark: 10 });

// Настройка стратегии по размеру в байтах
const byteStrategy = new ByteLengthQueuingStrategy({ highWaterMark: 1024 * 1024 }); // 1 МБ

const readableStream = new ReadableStream(
  {
    start(controller) {
      // ...
    }
  },
  strategy // стратегия управления очередью
);
```

**Пример с проверкой `desiredSize`:**

```javascript
const writableStream = new WritableStream({
  async write(chunk) {
    // медленный потребитель
    await slowProcessing(chunk);
  }
});

const writer = writableStream.getWriter();
for (const chunk of largeDataSet) {
  // Проверяем backpressure
  if (writer.desiredSize <= 0) {
    // Поток заполнен — ждем освобождения
    await writer.ready;
  }
  await writer.write(chunk);
}
```

### 2.2. Проблемы игнорирования Backpressure

Игнорирование сигналов backpressure приводит к неконтролируемому росту очереди и исчерпанию памяти. Типичные сценарии:

- **Производитель не проверяет `desiredSize` или `writer.ready`**.
- **Вложенные TransformStream без корректной передачи backpressure** — каждый TransformStream должен синхронизировать свой приемник и источник.
- **Необработанные ветвления** — при `tee()` (разветвлении потока) backpressure агрегируется по всем ветвям; если одна ветвь читается медленно, весь поток замедляется.

```javascript
// НЕПРАВИЛЬНО: игнорирование backpressure
const writer = writableStream.getWriter();
for (const chunk of largeDataSet) {
  writer.write(chunk); // ошибка: не проверяется writer.ready
}

// ПРАВИЛЬНО: проверка backpressure
const writer = writableStream.getWriter();
for (const chunk of largeDataSet) {
  await writer.ready; // ждем, пока есть место
  await writer.write(chunk);
}
```

### 2.3. Использование `pipeTo` и `pipeThrough` для автоматического управления Backpressure

Методы `pipeTo` и `pipeThrough` автоматически управляют backpressure между источником и приемником. Это самый простой и надежный способ построения потоковых конвейеров.

```javascript
// Автоматическое управление backpressure
await readableStream.pipeTo(writableStream);
```

---

## 3. Fetch + Streams: рендеринг HTML по частям (Streaming SSR)

Одно из самых мощных применений Streams API в 2026 году — **Streaming Server-Side Rendering (SSR)**, когда сервер отправляет HTML клиенту по частям, по мере готовности, а не целиком после завершения рендеринга всего документа.

### 3.1. Как работает Streaming SSR

Традиционный SSR требует полной загрузки всех данных и рендеринга всей страницы на сервере до отправки ответа клиенту. При подходе со стримингом:

1. Сервер немедленно отправляет начальную часть HTML — `<head>`, `<body>` и оболочку страницы.
2. Сервер продолжает рендерить отдельные компоненты страницы по мере загрузки их данных.
3. Каждый готовый компонент немедленно отправляется клиенту в виде HTML-фрагмента.
4. Браузер получает и отображает чанки по мере поступления — пользователь видит страницу раньше, чем она полностью загружена.
5. После завершения передачи страница гидратируется (если используется React или другой фреймворк).

### 3.2. Практическая реализация с React

В React 19+ `renderToReadableStream()` позволяет генерировать поток HTML и отправлять его клиенту. В сочетании с `<Suspense>` компоненты могут стримиться по мере загрузки данных.

```javascript
import { renderToReadableStream } from 'react-dom/server';

async function handleRequest(req, res) {
  const stream = await renderToReadableStream(
    <App />,
    {
      onError(error) {
        console.error(error);
      }
    }
  );

  // Отправка потока клиенту
  const response = new Response(stream, {
    headers: { 'Content-Type': 'text/html' }
  });
  return response;
}
```

```jsx
// Компонент, который может стримиться
function ProductPage({ productId }) {
  return (
    <Suspense fallback={<div>Загрузка товара...</div>}>
      <ProductDetails id={productId} />
    </Suspense>
  );
}

async function ProductDetails({ id }) {
  const product = await fetchProduct(id); // асинхронная загрузка
  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
    </div>
  );
}
```

### 3.3. Особенности Streaming SSR в 2026 году

**1. Стриминг и SEO-метаданные.** Современные фреймворки решают проблему, когда метаданные для SEO (`<title>`, `<meta>`), загружаемые асинхронно, не попадают в исходный HTML. **Next.js 16** здесь — пожалуй, наиболее продуманная реализация среди фреймворков: он определяет ботов по User-Agent (список `htmlLimitedBots`, включающий Googlebot, Bingbot, а также социальные краулеры вроде Twitter/Facebook/LinkedIn/Slack/Discord) и для них отключает стриминг метаданных, отдавая полный блокирующий `<head>` сразу. Для обычных пользователей `generateMetadata()` остаётся неблокирующим: метаданные дописываются позже через `<script>`-инструкции, которые клиентский рантайм переносит в `<head>`.

```javascript
// Next.js 16 — бот-осведомленный стриминг
export async function generateMetadata({ params }) {
  const { id } = await params;
  const product = await fetch(`/api/products/${id}`).then(r => r.json());
  return {
    title: product.name,
    openGraph: { images: [product.image] }
  };
}
// Для реальных пользователей — стриминг.
// Для ботов из списка htmlLimitedBots — полный блокирующий HTML.
```

**2. Поддержка в фреймворках (2026).** Экосистема здесь быстро меняется от релиза к релизу, поэтому конкретные детали ниже стоит перепроверять по актуальной документации фреймворка перед принятием архитектурных решений:
- **Next.js 16** — бот-осведомленный стриминг (по умолчанию для пользователей, блокировка для ботов из `htmlLimitedBots`), подтверждённый выше.
- **Remix** — стриминг, но `meta()` функция блокирует head, поэтому метаданные всегда полные.
- **Astro** — стриминг, но только после завершения рендеринга фронтматтера.
- **Nuxt 4** — блокирующий SSR (вся страница буферизируется).

**3. Ограничения и подводные камни:** 
- Нельзя использовать хуки, которые выполняются на клиенте (`useEffect` с запросами), внутри стриминг-компонента — это обнуляет выгоду от стриминга, так как данные запрашиваются уже после гидратации на клиенте.
- Условный рендеринг на основе `window` или `document` внутри стриминг-компонента вызывает расхождение (mismatch) между SSR и CSR.
- Live Preview на серверной стороне плохо сочетается со стримингом.

### 3.4. Потоковая передача файлов и больших ответов

Streaming SSR — частный случай использования Fetch + Streams. Более общий подход — потоковая передача любых данных.

```javascript
async function handleRequest(req) {
  // Создаем поток ответа
  const stream = new ReadableStream({
    async start(controller) {
      // Пишем заголовок
      controller.enqueue(new TextEncoder().encode('<html><body>'));

      // Получаем данные по частям
      for await (const chunk of fetchLargeDataset()) {
        const htmlChunk = renderChunk(chunk);
        controller.enqueue(new TextEncoder().encode(htmlChunk));
      }

      // Завершаем документ
      controller.enqueue(new TextEncoder().encode('</body></html>'));
      controller.close();
    }
  });

  return new Response(stream, {
    headers: { 'Content-Type': 'text/html' }
  });
}
```

*(здесь `fetchLargeDataset()` — предполагается собственный async-генератор на стороне сервера, а не сама браузерная async-итерация ReadableStream — на сервере в Node.js/Deno такие ограничения по браузерной поддержке не действуют)*

---

## Итог

Streams API — это фундаментальная технология 2026 года, которая позволяет обрабатывать данные любого размера с минимальным потреблением памяти и высокой производительностью. **ReadableStream, WritableStream и TransformStream** образуют единую модель для потоковой обработки, а механизм **backpressure** обеспечивает управление потоком данных, предотвращая переполнение памяти. Стоит помнить, что базовые операции потоков — Baseline и кросс-браузерны, а вот удобная async-итерация (`for await...of` прямо на потоке) пока работает только в Chrome, Edge и Firefox — для Safari нужен fallback через `reader.read()`.

**Streaming SSR** с использованием `fetch()` + `ReadableStream` позволяет отправлять HTML по частям, сокращая время до первого байта и улучшая воспринимаемую производительность. Современные фреймворки (Next.js, Remix, Astro, Nuxt) по-разному балансируют между стримингом и SEO — Next.js 16 с его бот-осведомлённым подходом сейчас выглядит наиболее проработанным решением, но экосистема меняется быстро, и стоит сверяться с актуальной документацией.

В следующей главе мы перейдем к аппаратному ускорению и рассмотрим **WebGPU** — современный стандарт для 3D-графики и высокопроизводительных вычислений в браузере.

---

*Перейти к следующей главе: [Глава 6. WebGPU — графический движок будущего](./chapter-06.md)*
