# MyCharacter

> Интерактивные листы персонажей НРИ с совместным редактированием, автосохранением и безопасным AI-ассистентом.

MyCharacter — RU/EN веб-приложение для хранения и заполнения интерактивных AcroForm PDF. Пользователь может загрузить собственный лист, редактировать поля прямо поверх PDF, пригласить соавтора, применить предложенные AI-изменения и скачать готовый интерактивный или flattened PDF.

## Возможности

| Область | Что поддерживается |
| --- | --- |
| Персонажи | Создание из собственного PDF или шаблона, переименование, клонирование и 30-дневная корзина |
| PDF-редактор | Text, multiline, checkbox, radio, dropdown и list-поля поверх PDF.js |
| Сохранение | Debounce 500 мс, немедленное сохранение на blur, версии полей и flush перед экспортом |
| Совместная работа | Realtime-синхронизация, presence, роли владельца и редактора, одноразовые приглашения |
| Каталогизация | PDF text layer, геометрические эвристики, OCR RU+EN и опциональный vision fallback |
| AI-ассистент | Персональная история диалогов и preview изменений с обязательным подтверждением |
| Экспорт | Интерактивный и flattened PDF с поддержкой кириллицы через Noto Sans |
| Локализация | Русский и английский интерфейс |

## Как устроен проект

```text
Browser
  ├─ Next.js App Router ── Auth, Dashboard, Route Handlers
  ├─ PDF.js + React ────── PDF и интерактивные контролы
  └─ CopilotKit / AG-UI ── AI-диалог и карточки предложений
           │
           ├─ Supabase ─── Auth, PostgreSQL, RLS, Storage, Realtime
           ├─ Inngest ──── каталогизация и фоновые задания
           └─ AI API ───── OpenAI-compatible chat/vision provider
```

Основные технологии:

- Next.js 16, React 19, TypeScript, Tailwind CSS и `next-intl`;
- Supabase Auth, PostgreSQL, Row Level Security, приватный Storage и Realtime;
- PDF.js, `pdf-lib`, `fontkit` и Tesseract.js;
- Inngest;
- CopilotKit v2, AG-UI и AI SDK.

## Требования

Проект запускается только через Docker. На хосте нужны:

- Docker Engine или Docker Desktop;
- Docker Compose v2;
- Docker Buildx.

Локальные Node.js, pnpm, Supabase CLI, PostgreSQL, Chromium и Inngest не требуются.

## Быстрый старт

### 1. Подготовьте конфигурацию

```bash
cp .env.example .env.local
cp .env.migrate.example .env.migrate.local
```

Заполните `.env.local`:

| Переменная | Назначение |
| --- | --- |
| `NEXT_PUBLIC_SUPABASE_URL` | URL проекта Supabase |
| `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` | Публичный ключ формата `sb_publishable_…` |
| `SUPABASE_SECRET_KEY` | Серверный ключ формата `sb_secret_…` |
| `AI_BASE_URL` | OpenAI-compatible API endpoint |
| `AI_API_KEY` | Серверный ключ AI-провайдера |
| `AI_CHAT_MODEL` | Модель со streaming и tool calls |
| `AI_VISION_MODEL` | Модель с поддержкой изображений |
| `NEXT_PUBLIC_APP_URL` | Публичный URL приложения, локально `http://localhost:3000` |

Legacy Supabase-ключи `anon` и `service_role` намеренно не поддерживаются. Секретные значения не должны попадать в браузер, логи или Git.

Для миграций укажите `SUPABASE_DB_URL` в `.env.migrate.local`. Используйте Direct connection или Session pooler URI из Supabase; пароль в URI должен быть URL-encoded.

### 2. Примените миграции

```bash
docker compose --profile tools run --rm migrate
```

Миграции создают таблицы, RPC, RLS-политики, Realtime publication и приватный bucket `character-pdfs`. Применённые production-миграции не следует переписывать — изменения схемы добавляются новым timestamped SQL-файлом.

### 3. Запустите development stack

```bash
docker compose up --build
```

После успешного healthcheck доступны:

- приложение — [http://localhost:3000](http://localhost:3000);
- Inngest Dev Server — [http://localhost:8288](http://localhost:8288);
- healthcheck — [http://localhost:3000/api/health](http://localhost:3000/api/health).

Исходники подключены как bind mount. `node_modules` и `.next` хранятся в Docker volumes, поэтому hot reload работает без установки зависимостей на хост.

Если порты заняты:

```bash
APP_PORT=3010 INNGEST_PORT=8388 INNGEST_CONNECT_PORT=8389 docker compose up --build
```

Остановить stack:

```bash
docker compose down
```

Команда `docker compose down -v` также удаляет контейнерные зависимости и кэш проекта. Используйте её только когда действительно нужна полная пересборка.

## Работа с PDF

Поддерживаются AcroForm PDF размером до 25 МБ и до 20 страниц. Импорт отклоняет:

- файлы без сигнатуры `%PDF-`;
- зашифрованные документы;
- XFA-only документы;
- PDF без AcroForm-полей.

Каталогизация выполняется в Inngest:

1. PDF.js извлекает поля, widgets, значения, типы, страницы и координаты.
2. Text layer и геометрический matcher определяют подписи, разделы и группы.
3. Для слабого text layer запускается Tesseract RU+EN.
4. При согласии пользователя и низкой уверенности используется vision-модель.
5. AI-ответ проверяется Zod-схемой; сбой vision оставляет результат в статусе `partial`.

## Безопасность AI

Ассистенту доступны только три серверных инструмента:

- `searchFields`;
- `getFieldContext`;
- `proposeFieldChanges`.

AI не записывает значения напрямую. Он формирует персональное предложение, пользователь выбирает изменения, а сервер применяет их отдельной PostgreSQL-транзакцией с повторной проверкой:

- доступа к персонажу;
- типа поля и допустимых options;
- ожидаемой версии;
- конфликтов параллельного редактирования.

## Курируемые шаблоны

PDF-шаблоны не включены в репозиторий из-за лицензионных ограничений. Разрешённый правообладателем файл можно импортировать административной командой:

```bash
docker compose exec -T app pnpm template:import ./sheet.pdf "Название шаблона" "Игровая система"
```

Файл должен находиться внутри репозитория, подключённого в контейнер как `/app`.

## Проверки

Минимальный набор после изменения кода:

```bash
docker compose exec -T app pnpm lint
docker compose exec -T app pnpm typecheck
docker compose exec -T app pnpm test
```

Изолированный check-контейнер:

```bash
docker compose --profile test run --rm check
```

Production build:

```bash
docker compose --profile test run --rm -e NODE_ENV=production check pnpm build
```

Полный E2E с Chromium:

```bash
docker compose --profile test run --rm e2e
```

## Production

Production Compose использует standalone Next.js runtime:

```bash
docker compose --env-file .env.local -f compose.prod.yaml build
docker compose --env-file .env.local -f compose.prod.yaml up -d
```

Перед публикацией:

1. примените актуальные Supabase migrations;
2. настройте Site URL и `/auth/callback` в Supabase Auth;
3. создайте Inngest app и синхронизируйте `/api/inngest`;
4. передайте runtime secrets через защищённое хранилище окружения;
5. проверьте `/api/health`, вход, загрузку PDF, каталогизацию, сохранение и экспорт.

Публичные Supabase-переменные передаются как build arguments, потому что Next.js встраивает их в клиентский bundle. `SUPABASE_SECRET_KEY`, AI API key и Inngest keys остаются только runtime-переменными.

## Структура репозитория

```text
src/app                  страницы и Route Handlers
src/components/editor    PDF-редактор, каталог и AI-sidebar
src/lib/ai               provider, runner, история и инструменты
src/lib/pdf              извлечение, OCR, каталогизация и экспорт
src/lib/supabase         browser/server/admin clients и auth helpers
src/inngest              фоновые функции
supabase/migrations      схема, RPC, RLS и Realtime
messages                 RU/EN-переводы
scripts                  Docker-миграции и импорт шаблонов
tests/e2e                Playwright-сценарии
```

## Лицензии и данные

Не добавляйте сторонние листы D&D, Pathfinder и других систем без разрешения правообладателя. Пользовательские PDF хранятся только в приватном bucket и выдаются через временные signed URLs после серверной проверки доступа.
