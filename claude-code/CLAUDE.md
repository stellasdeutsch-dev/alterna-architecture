# Alterna — онлайн-платформа турецкого языка

Claude, читай этот файл в начале каждой сессии. Если задача противоречит правилам ниже — остановись и спроси.

## Продукт

- Веб-платформа по подписке. Только браузер: телефон и компьютер. Мобильного приложения нет и не планируется.
- Язык интерфейса — русский. Курс — турецкий A1–C1.
- Маршрут из 5 городов, уровень = город: A1 Стамбул, A2 Измир, B1 Анталья, B2 Анкара, C1 Самсун.
- 310 тем = 310 гайдов: 158 грамматика + 152 лексика. Источник истины — `content/maps/*.json`.
- Id темы = id гайда: `a1.dative` (грамматика), `v.a1.cafe` (лексика).
- Роль `app_role`: `user` | `admin` | `superadmin`.
- «Пробный» и «ученик» — не роли, а статус доступа `trial` | `student` | `expired`. Считается в `src/domain/access.ts`.
- Суперадмин один — владелец. Назначается только SQL-миграцией, из интерфейса — никогда.

## Стек (версии на октябрь 2026)

| Слой | Что | Версия |
|---|---|---|
| Фреймворк | Next.js App Router, React, TypeScript strict | 16.4 · 19.3 |
| Стили | Tailwind CSS + shadcn/ui | 4.3 · CLI 4.21 |
| UI-библиотеки | Motion, Recharts, Zustand, zod, react-hook-form | 14 · 3.10 · 5 · 4.6 |
| Бэкенд | Supabase: Postgres, Auth, Storage, pg_cron | supabase-js 2.117 · @supabase/ssr 0.12 |
| Оплата | Paddle Billing: @paddle/paddle-js, @paddle/paddle-node-sdk | 1.6 · 3.10 |
| Повторение слов | ts-fsrs | 5.4 |
| Гайды | HTML + CSS + GSAP | 3.15 |
| Письма | Resend + React Email | 6.32 |
| Мониторинг | @sentry/nextjs, posthog-js | 11.6 · 1.438 |
| Тесты, линтер | Vitest, Playwright, Biome | 5.0 · 1.64 |
| Озвучка | Azure Speech: tr-TR-EmelNeural, tr-TR-AhmetNeural | — |
| Хостинг | Vercel Pro, функции в `fra1`; Supabase Pro, `eu-central-1` (Франкфурт) | — |

## Новые API — не пиши по старым туториалам

- **Next.js 16:** файл `proxy.ts`, а не `middleware.ts`; экспортирует функцию `proxy`.
- **Кэш:** `cacheComponents: true` в `next.config.ts`, директива `"use cache"`, `cacheTag()` и `revalidateTag()`. Turbopack включён по умолчанию.
- **Ключи Supabase:** `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` (`sb_publishable_…`) и `SUPABASE_SECRET_KEY` (`sb_secret_…`). Ключи anon и service_role не используем — их отключают в конце 2026.
- **Сессия:** `@supabase/ssr`, `createServerClient` с cookies.
- **Пользователь на сервере:** `supabase.auth.getClaims()` — асимметричные JWT, проверка локально. `getSession()` на сервере не доверяем.
- **Роль в JWT:** Custom Access Token Hook добавляет claim `app_role`. В RLS: `((select auth.jwt()) ->> 'app_role')`.
- **RLS:** `auth.uid()` и `auth.jwt()` всегда оборачиваем в `(select …)` — так Postgres вычисляет их один раз на запрос.
- **GSAP:** все плагины бесплатны с апреля 2025, ставим из npm-пакета `gsap`. `gsap-trial` не используем.

## Структура

```
alterna/
  src/
    app/                 # маршруты Next.js
      (marketing)/       # лендинг, тарифы, оферта, политика, возвраты
      (app)/             # план, дашборд, слова, библиотека, игры, настройки
      learn/[lessonId]/  # плеер гайда и уроков
      admin/  superadmin/
      g/[topic]/[version]/route.ts   # отдаёт гайд после проверки доступа
      api/track/  api/v1/  api/webhooks/paddle/  api/health/
    domain/              # чистая логика + тесты: access, unlock, xp, hours, fsrs
    content/schema/      # zod: карты, guide.json, упражнения, game.json
    lib/                 # supabase (server/client), paddle, resend, posthog
    components/          # shadcn/ui + свои
  public/
    kit/v1/              # kit.css, kit.js, tracker.js, game-sdk.js, gsap.min.js
    games/<id>/          # index.html, game.json
  guides/
    _template/           # эталонный шаблон гайда
    a1-istanbul/<topic>/ # index.html, guide.json, media/
  content/
    maps/                # alterna-grammar-a1-c1.json, alterna-vocabulary-a1-c1.json
    exercises/<topic>.json
  supabase/
    migrations/  seed.sql
  scripts/               # build-plan.ts, content-publish.ts, content-sync.ts
  tests/                 # e2e (Playwright), rls
  .github/workflows/ci.yml
```

## Правила

1. Схему БД меняем только миграциями в `supabase/migrations`. Прод руками не трогаем.
2. RLS включён на каждой таблице. Секретный ключ используем только на сервере (`import 'server-only'`), в скриптах и CI.
3. Прогресс, XP, доступ и подписки пишет только сервер. У клиента нет политик insert и update на эти таблицы.
4. Бизнес-логика живёт в `src/domain` как чистые функции с тестами Vitest. UI и API только вызывают их.
5. «Урок пройден → XP → стрик → следующий урок» — одна Postgres-функция, вызов через RPC с сервера.
6. Контент живёт в Git. Таблицы каталога перезаписывает `pnpm content:sync`, руками в базе их не редактируем.
7. Mobile-first: каждый экран проверяй на 375 px — без горизонтального скролла.
8. Тексты интерфейса — по-русски, коротко, без канцелярита.
9. Зависимости без причины не добавляем. Файлы вне задачи не трогаем. Коммиты маленькие.
10. Вебхуки Paddle проверяем по подписи и обрабатываем идемпотентно по `event_id`.
11. Секреты не коммитим. `.env.local` в `.gitignore`, пример переменных — в `.env.example`.

## Маршрут и план

`scripts/build-plan.ts` собирает каждый город в таком порядке:

1. «Прибытие» — только A2 и B2.
2. Грамматика — топологическая сортировка по `prerequisites`.
3. Лексика — после своей `recommended_after_grammar`.
4. Юниты по 5–7 уроков, к каждому — практика и тест юнита.
5. «Остановка», «Инструменты словаря», тест уровня.

Правила открытия уроков:

| Что | Правило |
|---|---|
| Грамматика | Жёстко: тема открыта, когда пройдены пререквизиты |
| Лексика | Мягко: рекомендация, без блокировки |
| «Прибытие» и повторение | Открыты всегда |
| Следующий город | Открывается тестом уровня |

Тема без опубликованного гайда показывается как «скоро» и не блокирует юнит.

## Контракт гайда

- Папка `guides/<город>/<topic>/`: `index.html` (один файл, CSS и JS внутри), `guide.json`, `media/`.
- Секции размечаются так: `<section data-section="ключ" data-words="N">`. Ключи стабильные и никогда не переиспользуются.
- Подключение kit:
  - `/kit/v1/kit.css`;
  - `/kit/v1/kit.js` (`type="module"`);
  - `/kit/v1/tracker.js` (`defer`);
  - `/kit/v1/gsap.min.js`.
- Озвучка: `<a-say>турецкий текст</a-say>`. Аудио генерирует `content:publish`.
- Упражнения: `<a-exercise ref="a1.dative.ex2">`. Данные лежат в `content/exercises/<topic>.json`, у каждого упражнения есть `topic` и `skill`.
- Бюджет гайда:
  - JS + CSS — не больше 400 КБ без медиа;
  - на 375 px нет горизонтального скролла;
  - в консоли нет ошибок.
- `guide.json` должен проходить схему `src/content/schema/guide.ts`.
- Публикация:
  - HTML → приватный бакет `guides`, путь `<topic>/v<version>/index.html`;
  - медиа → публичный бакет `media`, имя файла = хеш содержимого.

## Доступ к гайду

iframe запрашивает `GET /g/<topic>/<version>` на том же домене. Route Handler работает так:

1. `getClaims()` → `access(user)` → проверяет, открыт ли урок.
2. Скачивает HTML: `storage.from('guides').download()`.
3. Отвечает `text/html` с заголовками:
   - `Cache-Control: private, max-age=3600`;
   - `Content-Security-Policy: frame-ancestors 'self'`.
4. Без доступа отвечает 403, приложение показывает пейволл.

## Трекинг

- **tracker.js:**
  - IntersectionObserver с `rootMargin: '-45% 0px -45% 0px'`;
  - активное время считается, только когда вкладка видна и ученик что-то делал за последние 30 с;
  - события отправляются так: `window.parent.postMessage({ source: 'alterna-guide', … }, location.origin)`.
- **Приложение:**
  - проверяет `event.origin` и `event.source`;
  - копит события и шлёт батч на `/api/track` раз в 30 с;
  - на `pagehide` отправляет остаток через `navigator.sendBeacon`.
- **События:** `guide_open`, `section_read`, `heartbeat`, `exercise_answer`, `audio_play`, `guide_complete`. У каждого есть uuid `id` для идемпотентности.
- **Секция прочитана:** она в фокусе и активное время ≥ 30% от `words / 150` минут.
- **Гайд пройден:** прочитано ≥ 80% секций и сдан тест.
- **Хранение:** `events` — партиции по месяцам, 90 дней. `heartbeat` в `events` не пишем, только увеличиваем `active_seconds`.

## Игры

- Игра — это папка `public/games/<id>/` с `game.json`. Запускается в `<iframe sandbox="allow-scripts">` — без `allow-same-origin`.
- `game-sdk.js`:
  - `AlternaGame.init()` ждёт контент от платформы через postMessage;
  - `score(n)` — отправить очки;
  - `complete({ correct, total })` — завершить игру.
- XP за игры начисляет сервер, с дневным лимитом из `game.json`.

## Подписка

- **Оплата:** Paddle Billing. Checkout — Paddle.js overlay, в `customData` передаём `user_id`.
- **Вебхуки** приходят на `/api/webhooks/paddle` и обновляют `subscriptions` и `payments`:
  - `subscription.created`, `subscription.updated`, `subscription.canceled`;
  - `transaction.completed`, `transaction.payment_failed`.
- **Пробный период** считает платформа: `profiles.trial_ends_at` = регистрация + 7 дней, без карты. Длина — из `settings`.
- **Статус доступа:**
  - `student` — подписка `active`, `past_due` или отменена, но период ещё не закончился;
  - `trial` — `trial_ends_at > now()`;
  - иначе `expired`.
- **Цены** хранятся в Paddle, в базе — только их id.

## Админка

- `/admin` — для admin и superadmin; `/superadmin` — только для superadmin. Проверка и в `proxy.ts`, и в каждом серверном действии.
- Для admin и superadmin обязателен MFA (TOTP): требуем уровень `aal2`.
- Каждое действие пишется в `audit_log`: кто, что, над кем, когда, значения до и после.

## Команды

```
pnpm dev | build | test | e2e | lint | typecheck
pnpm gen:types                  # supabase gen types → src/lib/database.types.ts
supabase start | stop | db reset | migration new <name> | db push
pnpm plan:build                 # карты → план городов
pnpm content:check <topic>      # контракт, схема, 375 px, вес
pnpm content:publish --changed  # озвучка, загрузка в Storage
pnpm content:sync               # каталог → Postgres
```

## Перед запуском

- [ ] RLS-тесты зелёные: ученик не видит чужое, пробный не открывает закрытый гайд, admin не назначает роли
- [ ] E2E на 375 px: регистрация → пробный → гайд → упражнение → оплата (sandbox) → Измир
- [ ] k6: `/api/track` при 1 000 виртуальных учеников, 95% ответов < 300 мс
- [ ] Вебхуки Paddle: повтор события не меняет данные дважды
- [ ] Секретный ключ не попадает в клиентский бандл (`grep sb_secret .next/static` пусто)
- [ ] MFA включён у суперадмина
- [ ] Бэкапы Supabase включены, восстановление проверено на staging
- [ ] Sentry и uptime-алерты приходят в Telegram
- [ ] Оферта, политика конфиденциальности и правила возврата опубликованы
