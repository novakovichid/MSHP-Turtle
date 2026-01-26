# Техническое руководство

## Обзор
MSHP-IDE — это статическое одностраничное приложение (SPA), которое запускает Python через Pyodide в Web Worker. Основной поток обрабатывает интерфейс, хранилище и отрисовку Turtle. Сервис-воркер добавляет заголовки COOP/COEP для опциональной изоляции cross-origin.

Проект также содержит вариант на базе Skulpt (`skulpt.html`) с полностью отдельными путями кода и ресурсами.

## Структура каталогов
- `index.html`: Оболочка SPA Pyodide и CSP.
- `skulpt.html`: Оболочка SPA Skulpt и CSP.
- `assets/app.js`: Логика интерфейса и выполнения Pyodide (роутер, редактор, хранилище, шеринг).
- `assets/skulpt-app.js`: Логика интерфейса и выполнения Skulpt (независимая копия).
- `assets/worker.js`: Воркер выполнения Pyodide.
- `assets/styles.css`: Стили Pyodide.
- `assets/skulpt-styles.css`: Стили Skulpt (независимая копия).
- `assets/fflate.esm.js`: Резервный вариант gzip для браузеров без CompressionStream.
- `assets/skulpt-fflate.esm.js`: Резервный вариант gzip для Skulpt (независимая копия).
- `pyodide-0.29.1/pyodide/`: Локальная среда выполнения Pyodide.
- `vendor/skulpt/skulpt-dist-master/`: Среда выполнения Skulpt и стандартная библиотека.
- `sw.js`: Сервис-воркер COI.

## Процесс загрузки
1. `index.html` загружает `assets/app.js` как модуль.
2. `init()` привязывает обработчики интерфейса, регистрирует сервис-воркер и проверяет поддержку среды (Worker и WebAssembly).
3. `ensureRuntimeCompatibility()` инициирует разовую перезагрузку, если требуется COI.
4. Открывается IndexedDB; при блокировке используется резервное хранилище в памяти.
5. Создается и инициализируется Web Worker с Pyodide.

## Роутинг
Используется роутинг на основе хеша (поддерживается GitHub Pages):
- `/#/` : главная страница.
- `/#/p/{projectId}` : редактируемый проект.
- `/#/s/{shareId}?p={payload}` : неизменяемый снимок.
- `/#/embed` : режим встраивания с настройками в строке запроса.

## Модель хранения
База данных IndexedDB: `mshp-ide` (Pyodide)
База данных IndexedDB: `mshp-ide-skulpt` (Skulpt)
Хранилища объектов:
- `projects`: `{ projectId, title, files, assets, lastActiveFile, updatedAt }`
- `blobs`: `{ blobId, data }`
- `drafts`: `{ key, overlayFiles, deletedFiles, draftLastActiveFile, updatedAt }`
- `recent`: `{ key: "recent", list: [projectId...] }`

Если IndexedDB недоступна, используется хранилище в оперативной памяти на базе `Map`.

## Воркер выполнения
`assets/worker.js`:
- `importScripts()` загружает Pyodide из `indexURL`.
- `loadPyodide()` инициализирует среду выполнения.
- Stdout/stderr проксируются в основной поток.
- `input()` реализован через очередь в основном потоке.
- Каждый запуск сбрасывает `/project` в файловой системе Pyodide и записывает файлы/ресурсы.
- Точка входа запускается через `runpy.run_path(..., run_name="__main__")`.

## Конвейер Turtle
- `TURTLE_SHIM` заменяет стандартный модуль `turtle` в воркере.
- События Turtle сериализуются и отправляются в основной поток.
- Основной поток отрисовывает графику на холсте (canvas) с управлением скоростью и анимацией.
- События указателя/касания нормализуются и отправляются обратно в воркер.

## Шеринг и снимки
- При шеринге состояние проекта сериализуется в JSON и кодируется как UTF-8 байты.
- Сжатие данных:
  - Используется `CompressionStream("gzip")`, если доступно.
  - Резервный вариант — `gzipSync` из `fflate`.
- Кодирование: base64url с префиксом:
  - `g.<data>` для gzip.
  - `u.<data>` для обычного текста.
- `shareId`:
  - SHA-256 через `crypto.subtle`, если доступно.
  - Резервный вариант — хеш на основе FNV-1a.
- Снимки неизменяемы; локальные правки сохраняются в `drafts`.

## CSP и COI
- CSP (мета-тег в `index.html`) включает:
  - `script-src 'self' 'wasm-unsafe-eval' 'unsafe-eval'`
  - `worker-src 'self' blob:`
- `sw.js` внедряет заголовки COOP/COEP/CORP для изоляции cross-origin.

## Совместимость с браузерами
Резервные механизмы в `assets/app.js`:
- Полифиллы для TextEncoder/TextDecoder.
- Генерация UUID без `crypto.randomUUID`.
- `CompressionStream/DecompressionStream` -> `fflate` gzip.
- `crypto.subtle digest` -> FNV-1a.
- `navigator.clipboard` -> `document.execCommand`.
- IndexedDB -> хранилище `Map` в памяти.
- Pointer events -> обработчики мыши/касаний.
- Загрузка воркера через `fetch()` -> прямой URL воркера.

Поддерживаемые браузеры:
- Chrome
- Chromium
- Safari

Неподдерживаемые браузеры:
- Firefox
- Другие движки, отличные от Chromium/Safari.

## Лимиты и защитные механизмы
- `RUN_TIMEOUT_MS`: 10 секунд (мягкое прерывание, затем жесткая остановка).
- `MAX_OUTPUT_BYTES`: 2,000,000 байт.
- `MAX_FILES`: 30.
- `MAX_SINGLE_FILE_BYTES`: 50,000 байт.
- `MAX_TOTAL_TEXT_BYTES`: 250,000 байт.

## Обновление Pyodide
1. Замените содержимое `pyodide-0.29.1/pyodide` на новую версию.
2. Обновите `indexURL` в `assets/app.js` (инициализация воркера).
3. Убедитесь, что `pyodide.js` и `pyodide.asm.wasm` соответствуют новой версии.

## Переключение между страницами Pyodide и Skulpt (инструкции для ИИ-агента)
Если версия на Skulpt должна стать основной:
1. Создайте резервную копию `index.html` (опционально).
2. Замените `index.html` копией `skulpt.html`.
3. Убедитесь, что ссылки в шапке/на главной по-прежнему ведут на альтернативную страницу.
4. Оставьте `assets/app.js` и `assets/skulpt-app.js` нетронутыми для легкого отката.

Чтобы вернуться на Pyodide, восстановите оригинальный `index.html`.

## Локальная разработка
- Используйте статический сервер (`serve.bat` или любой HTTP-сервер).
- Этап сборки не требуется.
- Сервис-воркер можно отключить, удалив регистрацию `sw.js`.

## Известные ограничения
- Шеринг не включает ресурсы (assets).
- Снимки не зашифрованы; они закодированы и опционально сжаты.
- Внешние сетевые запросы заблокированы политикой CSP.
