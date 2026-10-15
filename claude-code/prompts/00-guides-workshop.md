# Промпт 00 — мастерская гайдов (запустить один раз)

Готовит фундамент для всех 310 гайдов: скрытый трекер, видео с YouTube, движок упражнений, шаблон, локальный «плеер платформы» и автопроверку.

- **Модель:** Opus 5.5, effort high, plan mode.
- **Где запускать:** в корне репозитория `alterna`. Позже туда же добавится платформа на Next.js.
- **Перед запуском:**
  - положи в корень `CLAUDE.md`;
  - карты грамматики и лексики — в `content/maps/`.

```
Подготовь мастерскую гайдов Alterna. По ней будут сделаны все 310 гайдов, а позже в этот же репозиторий добавится платформа на Next.js. Прочитай CLAUDE.md — разделы «Контракт гайда», «Трекинг», «Адаптивность». Сначала покажи план, жди моего «ок».

СТРУКТУРА (как в будущей платформе)
public/kit/v1/        kit.css, kit.js, tracker.js, gsap.min.js (из npm-пакета gsap)
guides/_template/     index.html, guide.json, exercises.json
guides/a1-istanbul/   сюда пойдут гайды A1
content/maps/         карты (уже лежат)
src/content/schema/   guide.ts, exercise.ts, events.ts (zod)
scripts/              dev-guides.mjs, guide-new.ts, guide-check.ts
tests/guides/
pnpm, TypeScript strict, Playwright, Vitest.

1. public/kit/v1/tracker.js — СКРЫТЫЙ ТРЕКЕР
- Без интерфейса и зависимостей, ≤ 8 КБ, подключается одной строкой.
- События по схеме events.ts (v: 1): guide_open, section_view, section_read, heartbeat, scroll_depth (25/50/75/100), exercise_start, exercise_answer, exercise_complete, quiz_complete, video_play, video_progress (25/50/75/100), video_complete, audio_play, guide_complete, guide_leave.
- Формат: { v, id (uuid), type, topic, guideVersion, sessionId, ts, data }. topic и guideVersion — из <meta name="alterna:topic"> и <meta name="alterna:version">.
- Секция в фокусе: IntersectionObserver, rootMargin '-45% 0px -45% 0px'. Активное время — только когда вкладка видна и было действие за последние 30 с.
- Секция прочитана: в фокусе и активное время ≥ 30% от data-words / 150 минут. % прочитанного — по словам секций. Гайд пройден: ≥ 80% и итоговый тест ≥ 60%.
- API для нестандартных упражнений: window.AlternaTracker.answer({exerciseId,itemId,correct,answer,timeMs}), .complete({exerciseId,correct,total}), .event(type,data), .state() (для отладки).
- Принимает от родителя { type: 'goto', section } — прокрутить к секции.
- Транспорты:
  a) внутри iframe платформы — postMessage({ source:'alterna-guide', events }, origin родителя) пачками раз в 30 с и на pagehide;
  b) открыт отдельно и у <script> есть data-endpoint — пачки через navigator.sendBeacon, с anonId из localStorage (try/catch);
  c) иначе — события только в памяти.
- ?debug=1 — скрытая панель поверх гайда: живой список событий, % прочитанного, активное время, результаты упражнений и видео. Без ?debug=1 ученик ничего не видит.

2. public/kit/v1/kit.js — WEB COMPONENTS (без фреймворка)
- <a-say>Arabaya bin.</a-say> — кнопка озвучки. Есть data-audio — mp3; иначе голос браузера tr-TR, скорость 0.9. Событие audio_play.
- <a-video yt="ID" start="сек" end="сек" title="…" channel="…">:
  - лёгкая обложка (картинка i.ytimg.com + кнопка), без загрузки YouTube до нажатия;
  - по нажатию — youtube-nocookie.com/embed/ID с enablejsapi=1, playsinline=1, rel=0, start, end + YouTube IFrame API;
  - события video_play, video_progress, video_complete (прогресс считается внутри фрагмента start–end);
  - 16:9 через aspect-ratio, на телефоне во всю ширину, на широком экране не шире 800 px;
  - если плеер не загрузился за 8 с — ссылка «Открыть на YouTube».
- <a-exercise ref="<topic>.exN"> и <a-quiz ref="<topic>.quiz">:
  - данные из ./exercises.json (fetch относительно гайда);
  - типы: gap_fill, multiple_choice, matching, build_sentence, true_false, sort_buckets, error_hunt;
  - мгновенная проверка, объяснение при ошибке, «попробовать ещё», прогресс внутри упражнения;
  - работает пальцем, мышью и клавиатурой (1–4 — выбор, Enter — проверить), aria-live, кнопки ≥ 44 px;
  - события exercise_start, exercise_answer, exercise_complete / quiz_complete.
- <a-word tr="elma" ru="яблоко"> — карточка слова с озвучкой.

3. public/kit/v1/kit.css — токены (цвета, шрифты, радиусы, отступы) и стили компонентов. Размеры через clamp(), mobile-first, prefers-reduced-motion.

4. СХЕМЫ (zod, src/content/schema)
- guide.ts: topic, code, city, cefr, kind (grammar|lexicon), version, status (draft|ready), title, sections[{ key, title, words, type? }], videos[{ yt, start, end, title, channel, section }], exercises[ids], quiz (id), vocab[].
- exercise.ts: общие поля id, topic, skill, type, difficulty (1–3), instruction, items[]; у каждого item — правильный ответ и explanation; тип custom — для нестандартных упражнений (items с ответами, логика в гайде).
- events.ts: все события из п. 1.

5. ШАБЛОН guides/_template — гайд на 5 секций, где есть все компоненты: <a-say>, <a-video>, 2 <a-exercise>, пример custom-упражнения через AlternaTracker, <a-quiz>. GSAP + ScrollTrigger, ScrollTrigger.refresh() при повороте экрана.

6. scripts/dev-guides.mjs (pnpm guides:dev) — локальный сервер:
- /kit/v1/* из public/kit/v1;
- /g/<topic>/ из guides/*/<topic>/;
- / — список тем A1 из карты по порядку (code, название, статус: нет / черновик / готов);
- /player?topic=<id> — имитация плеера платформы: гайд в iframe, сбоку панель входящих событий трекера, кнопка «Продолжить с секции». Так режим iframe проверяется уже сейчас.

7. scripts/guide-new.ts (pnpm guide:new <topic>) — копирует шаблон в guides/<город>/<topic>/, заполняет guide.json и <meta> из карты.

8. scripts/guide-check.ts (pnpm guide:check <topic>) — Playwright, отчёт в tests/guides/report:
- guide.json и exercises.json проходят схемы; data-section совпадают с guide.json; ключи уникальны; data-words отличается от реального числа слов не больше чем на 15%;
- каждое yt проверено через https://www.youtube.com/oembed?url=https://www.youtube.com/watch?v=ID&format=json (200 — можно встраивать);
- 360, 768, 1280, 1920 px в WebKit и Chromium: нет горизонтального скролла, нет ошибок консоли, скриншоты в отчёт;
- JS + CSS ≤ 400 КБ без медиа;
- трекер: в /player с ускоренными часами (page.clock) прокрутить все секции и пройти все упражнения правильными ответами — должны прийти section_read на каждую секцию, exercise_answer по каждому вопросу, quiz_complete, guide_complete и % прочитанного = 100.

9. guides/README.md — как создать, посмотреть и проверить гайд.

Готово, когда pnpm guide:check _template зелёный и /player?topic=_template&debug=1 показывает события. В конце покажи, как запустить.
```
