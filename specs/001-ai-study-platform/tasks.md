# Tasks: AI Study Platform — Core Application

**Branch**: `001-ai-study-platform`
**Input**: Design documents from `/specs/001-ai-study-platform/`
**Prerequisites**: plan.md ✅ spec.md ✅ research.md ✅ data-model.md ✅ contracts/openapi.yaml ✅ quickstart.md ✅

**Tests**: Не включены (не запрошены в спецификации).

**Organization**: Задачи сгруппированы по User Story для независимой реализации и тестирования.

## Format: `[ID] [P?] [Story?] Description`

- **[P]**: Можно запускать параллельно (разные файлы, нет незавершённых зависимостей)
- **[Story]**: К какой User Story относится задача (US1–US6)
- Все пути файлов — от корня репозитория

---

## Phase 1: Setup (Инициализация проекта)

**Цель**: Создать скелет репозитория и контейнерную инфраструктуру.

- [ ] T001 Создать `docker-compose.yml` с четырьмя сервисами (frontend, backend, worker, redis) и `docker-compose.override.yml` для hot-reload в dev-режиме
- [ ] T002 Создать `backend/pyproject.toml` с зависимостями (fastapi, sqlmodel, aiosqlite, arq, authlib, py-fsrs, PyMuPDF, pdfplumber, python-pptx, mammoth, faster-whisper, ffmpeg-python, easyocr, langchain-text-splitters, tiktoken, cryptography, portalocker, alembic, uvicorn) и `backend/Dockerfile`
- [ ] T003 [P] Создать `frontend/package.json` с зависимостями (svelte, @sveltejs/kit, vite, typescript) и `frontend/Dockerfile` на базе `@sveltejs/adapter-node`
- [ ] T004 [P] Инициализировать Alembic в `backend/migrations/` — создать `alembic.ini`, `env.py` с async-движком aiosqlite, папку `versions/`
- [ ] T005 [P] Создать `.env.example` с переменными: SECRET_KEY, MAX_FILE_SIZE_BYTES, WHISPER_MODEL, REDIS_URL, DATA_DIR, CORS_ORIGINS

**Checkpoint**: `docker compose build` завершается без ошибок; структура директорий создана.

---

## Phase 2: Foundational (Блокирующие зависимости)

**Цель**: Общая инфраструктура, которая ОБЯЗАНА быть готова до начала любой User Story.

**⚠️ КРИТИЧНО**: Ни одна User Story не может быть начата до завершения этой фазы.

- [ ] T006 Реализовать `backend/src/config.py` — все настройки из переменных окружения через Pydantic Settings: пути к данным, лимиты файлов, модель Whisper, URL Redis, CORS origins, версия API
- [ ] T007 [P] Реализовать `backend/src/utilities/error_utils.py` — стандартизация `ErrorResponse {error, message, details}`, структурированное логирование через `logging`, декоратор для перехвата исключений в роутерах
- [ ] T008 [P] Реализовать `backend/src/utilities/crypto_utils.py` — Fernet-шифрование, PBKDF2 деривация ключа (1 200 000 итераций), хелперы encrypt/decrypt, генерация и чтение соли из `~/.config/study-kit/.salt`
- [ ] T009 [P] Реализовать `backend/src/utilities/file_utils.py` — определение MIME-типа через `python-magic`, белый список допустимых типов, конвенция пути `/data/uploads/{lesson_id}/{file_id}/{original_name}`, вычисление SHA-256 чексуммы
- [ ] T010 Реализовать `backend/src/database.py` — async SQLite движок (aiosqlite WAL), PRAGMA (`journal_mode=WAL`, `synchronous=NORMAL`, `foreign_keys=ON`, `busy_timeout=5000`), функция `get_session()` для FastAPI Depends, `init_db()`
- [ ] T011 [P] Создать `backend/src/models/workspace.py` — SQLModel `Workspace` с полями: id (UUID PK), name (TEXT NOT NULL, max 200), description (TEXT nullable), created_at, updated_at; валидация: name не пустое, case-insensitive unique
- [ ] T012 [P] Создать `backend/src/models/lesson.py` — SQLModel `Lesson` с полями: id, workspace_id (FK → workspaces), name, order_index (INTEGER default 0), created_at, updated_at; индекс `idx_lessons_workspace` ON (workspace_id, order_index)
- [ ] T013 [P] Создать `backend/src/models/agent_config.py` — SQLModel `AgentConfig` с полями: id, provider (TEXT UNIQUE: anthropic/openai), status (connected/expired/error/not_configured), provider_account_id (nullable), created_at, updated_at
- [ ] T014 [P] Создать `backend/src/models/source_file.py` — SQLModel `SourceFile` с полями: id, lesson_id (FK), original_name, mime_type, size_bytes, file_path, processing_status (queued/processing/completed/failed), error_message (nullable), uploaded_at, processed_at; индексы `idx_source_files_lesson`, `idx_source_files_status`
- [ ] T015 [P] Создать `backend/src/models/knowledge_item.py` — SQLModel `KnowledgeItem` с FSRS-полями: stability (REAL nullable), difficulty (REAL nullable), due_date (DATE), last_review_date (nullable), state (new/learning/review/relearning), step; агрегаты: total_reviews, correct_reviews, lapse_count, streak; индексы `idx_ki_lesson_due`, `idx_ki_lesson_state`, `idx_ki_due_state`
- [ ] T016 [P] Создать `backend/src/models/learning_session.py` — SQLModel `LearningSession` с полями: id, session_type (flashcards/test/chat), filter_config (JSON TEXT), started_at, ended_at (nullable), total_items, correct_items; индекс `idx_sessions_started`
- [ ] T017 [P] Создать `backend/src/models/performance_record.py` — SQLModel `PerformanceRecord` append-only: id, knowledge_item_id (FK), session_id (FK), reviewed_at, rating (1–4), was_correct, stability_before/after, difficulty_before/after, scheduled_interval_days, elapsed_days; индексы `idx_perf_item`, `idx_perf_session`
- [ ] T018 [P] Создать `backend/src/models/processing_job.py` — SQLModel `ProcessingJob`: id (TEXT PK = ARQ job id), source_file_id (FK), status, progress_pct (0–100), stage (extracting_text/chunking/generating_items nullable), created_at, updated_at
- [ ] T019 Создать Alembic миграцию `backend/migrations/versions/0001_initial.py` — все 8 таблиц с индексами из data-model.md; upgrade() и downgrade()
- [ ] T020 Реализовать `backend/src/main.py` — FastAPI app, lifespan (init_db при старте), подключение всех роутеров под `/api/v1`, CORS middleware из config, глобальный exception handler возвращающий `ErrorResponse`
- [ ] T021 [P] Создать `frontend/src/lib/components/ErrorMessage.svelte` — переиспользуемый компонент для отображения `ErrorResponse` с кодом и описанием ошибки

**Checkpoint**: `docker compose up` запускается; `GET http://localhost:8000/api/v1/docs` возвращает OpenAPI UI; `GET http://localhost:3000` возвращает пустой SvelteKit layout.

---

## Phase 3: User Story 1 — Подключение AI-агента (Priority: P1) 🎯 MVP

**Цель**: Пользователь подключает Anthropic (paste-flow) или OpenAI (PKCE OAuth2) и видит статус `connected` в шапке.

**Independent Test**: Вставить `sk-ant-...` ключ → статус меняется на `connected ✓`; зашифрованный файл появляется в `~/.config/study-kit/credentials.json.enc`; перезапуск приложения — статус сохраняется.

- [ ] T022 [US1] Реализовать `backend/src/services/auth_store.py` — Fernet-шифрование токенов (использует crypto_utils.py), portalocker для concurrent access, CRUD по `agentId` в `~/.config/study-kit/credentials.json.enc`, метод `load_token(provider)` / `save_token(provider, data)` / `delete_token(provider)`
- [ ] T023 [US1] Реализовать `backend/src/services/oauth_service.py` — paste-flow для Anthropic (валидация ключа через `GET https://api.anthropic.com/v1/models`, сохранение через auth_store); PKCE OAuth2 для OpenAI: генерация code_verifier (64 bytes base64url) + code_challenge (SHA-256), запуск локального callback-сервера на случайном порту, обмен code+verifier на токены, авто-refresh через `httpx`
- [ ] T024 [P] [US1] Реализовать `backend/src/services/ai_client.py` — единый клиент для Anthropic SDK и OpenAI SDK; метод `check_connectivity(provider)` для валидации токена; фабрика клиента по provider из auth_store; обработка `AuthenticationError` → обновление статуса в AgentConfig
- [ ] T025 [US1] Реализовать `backend/src/api/agent.py` — роутер `/agent`: `GET /agent/status` (список AgentStatus из AgentConfig), `POST /agent/connect/anthropic` (paste-flow через oauth_service), `GET /agent/connect/openai/start` (возвращает auth_url + callback_port), `GET /agent/connect/openai/callback` (завершает PKCE), `DELETE /agent/disconnect/{provider}` (удаляет токен и сбрасывает статус)
- [ ] T026 [P] [US1] Создать `frontend/src/lib/api/agent.ts` — TypeScript-клиент: `getAgentStatus()`, `connectAnthropic(apiKey)`, `startOpenAIOAuth()`, `disconnectAgent(provider)`; типы `AgentStatus`, `AgentProvider`
- [ ] T027 [P] [US1] Создать `frontend/src/lib/components/AgentStatusBadge.svelte` — Svelte 5 runes: отображает статус (connected ✓ / expired ⚠ / error ✗ / not_configured); polling каждые 30 с через `$effect`; кликабельная ссылка на `/settings`
- [ ] T028 [US1] Реализовать `frontend/src/routes/settings/+page.svelte` — страница настроек: карточка Anthropic (поле API-ключ + кнопка «Подключить»), карточка OpenAI (кнопка «Авторизовать через OAuth2»); stateful feedback через Svelte 5 `$state`; использует `ErrorMessage.svelte` для ошибок
- [ ] T029 [US1] Реализовать `frontend/src/routes/+layout.svelte` — глобальный layout: `AgentStatusBadge` в шапке, навигация (главная / дашборд / настройки), slot для страниц; onboarding-редирект на `/settings` если ни один агент не подключён

**Checkpoint**: Пользователь вставляет API-ключ Anthropic → статус в шапке меняется на `connected ✓`; повторный вход — статус сохранён.

---

## Phase 4: User Story 2 — Управление рабочими пространствами и уроками (Priority: P1)

**Цель**: Пользователь создаёт, переименовывает, удаляет рабочие пространства и уроки; данные сохраняются после перезагрузки.

**Independent Test**: Создать workspace «Сербский», добавить 2 урока, переименовать урок 1, удалить урок 2 → обновить страницу → все изменения сохранены.

- [ ] T030 [P] [US2] Реализовать `backend/src/api/workspaces.py` — роутер `/workspaces`: `GET /workspaces`, `POST /workspaces` (валидация: name не пустое, case-insensitive unique), `GET /workspaces/{id}`, `PATCH /workspaces/{id}`, `DELETE /workspaces/{id}` (каскадное удаление с явным подтверждением через `?confirm=true`); все ответы — схемы из openapi.yaml
- [ ] T031 [P] [US2] Реализовать `backend/src/api/lessons.py` — роутер `/workspaces/{id}/lessons` и `/lessons`: `GET /workspaces/{id}/lessons` (сортировка по order_index), `POST /workspaces/{id}/lessons` (auto-increment order_index), `GET /lessons/{id}`, `PATCH /lessons/{id}` (name, order_index), `DELETE /lessons/{id}` (каскадное удаление)
- [ ] T032 [P] [US2] Создать `frontend/src/lib/api/workspaces.ts` — TypeScript-клиент: `listWorkspaces()`, `createWorkspace(name, description?)`, `updateWorkspace(id, patch)`, `deleteWorkspace(id)`; тип `Workspace`
- [ ] T033 [P] [US2] Создать `frontend/src/lib/api/lessons.ts` — TypeScript-клиент: `listLessons(workspaceId)`, `createLesson(workspaceId, name)`, `updateLesson(id, patch)`, `deleteLesson(id)`; тип `Lesson`
- [ ] T034 [US2] Реализовать `frontend/src/routes/+page.svelte` — главный экран: список workspace-карточек, кнопка «Новое пространство» с inline-формой, пустое состояние с призывом к действию; данные загружаются через `+page.ts` load function
- [ ] T035 [US2] Реализовать `frontend/src/routes/workspaces/[id]/+page.svelte` — экран рабочего пространства: список уроков с порядком (drag или кнопки ↑↓), кнопка «Новый урок», rename/delete для каждого урока с подтверждением удаления; переход в урок по клику
- [ ] T036 [US2] Создать базовую структуру `frontend/src/routes/lessons/[id]/+page.svelte` — заголовок урока, хлебные крошки (workspace → lesson), placeholder для секций файлов и элементов знания (заполняется в US3)

**Checkpoint**: Полный CRUD workspaces и lessons работает через UI; обновление страницы сохраняет данные.

---

## Phase 5: User Story 3 — Загрузка и автоматическая обработка файлов (Priority: P1)

**Цель**: Пользователь перетаскивает файл → видит прогресс обработки → после завершения — список извлечённых элементов знания.

**Independent Test**: Загрузить PDF (до 50 МБ) и MP3 (до 30 мин) в урок → PDF обработан < 60 с, MP3 < 5 мин → в уроке видны knowledge items; при загрузке `.xyz` — ошибка «неподдерживаемый формат».

- [ ] T037 Реализовать `backend/src/processing/text_chunker.py` — обёртка над `RecursiveCharacterTextSplitter`; конфиги по типу: PDF/аудио → 512 токенов, 64 overlap; PPTX → граница слайда; DOCX (Markdown) → heading-разделители (`# `, `## `), затем recursive; аудио → 512 токенов, 128 overlap; метод `chunk(text, format) → list[str]`
- [ ] T038 [P] [US3] Реализовать `backend/src/processing/pdf_processor.py` — PyMuPDF для текстовых страниц + pdfplumber для таблиц; возвращает `list[str]` (текстовые блоки по страницам); логирует страницы с ошибкой OCR, не падает на них
- [ ] T039 [P] [US3] Реализовать `backend/src/processing/pptx_processor.py` — python-pptx; извлекает текст по слайдам (title + body shapes); каждый слайд → отдельный чанк; логирует слайды без текста
- [ ] T040 [P] [US3] Реализовать `backend/src/processing/docx_processor.py` — mammoth для конвертации DOCX → Markdown; затем text_chunker с heading-стратегией; обработка python-docx для структуры при наличии таблиц
- [ ] T041 [P] [US3] Реализовать `backend/src/processing/audio_processor.py` — ffmpeg-python для извлечения аудио из видео (MP4, MOV → WAV); faster-whisper транскрипция (модель из config.WHISPER_MODEL); возвращает текст транскрипции; логирует прогресс каждые 30 с
- [ ] T042 [P] [US3] Реализовать `backend/src/processing/image_processor.py` — easyocr как основной (устойчив к шуму); pytesseract как fallback для чистых сканов; Claude Vision API как последний резерв для сложных макетов (только если агент подключён); возвращает `str`
- [ ] T043 [US3] Реализовать `backend/src/processing/item_extractor.py` — вызов AI-агента (через ai_client.py) для каждого чанка; системный промпт: извлечь атомарные knowledge items в формате `{question, answer, context}`; retry 3× при rate limit; возвращает `list[KnowledgeItem]`
- [ ] T044 [US3] Реализовать `backend/src/processing/pipeline.py` — оркестратор: определяет MIME-тип → вызывает нужный processor → text_chunker → item_extractor; обновляет `ProcessingJob.stage` и `progress_pct` в процессе; при исключении → `status=failed`, `error_message`; при успехе → `source_file.processing_status=completed`, `processed_at=now()`
- [ ] T045 [US3] Реализовать `backend/src/processing/worker.py` — ARQ `WorkerSettings`: регистрация задачи `process_file(ctx, source_file_id)`; retry 2× при падении; max_jobs=4; timeout 600 с; Redis DSN из config
- [ ] T046 [US3] Реализовать `backend/src/api/files.py` — роутер: `GET /lessons/{id}/files`, `POST /lessons/{id}/files` (multipart: проверка MIME + размера, сохранение на ФС, создание SourceFile, enqueue ARQ job → возврат `{source_file, job_id}`), `DELETE /files/{id}` (удаление ФС + БД); `GET /jobs/{job_id}` (статус ProcessingJob из Redis/БД)
- [ ] T047 [US3] Реализовать `backend/src/api/items.py` — роутер: `GET /lessons/{id}/items` (пагинация page/page_size, фильтр по state); `GET /items/due` (по filter_config: mode, lesson_ids, limit); ответы — `PaginatedKnowledgeItems` и `list[KnowledgeItem]`
- [ ] T048 [P] [US3] Создать `frontend/src/lib/api/files.ts` — TypeScript-клиент: `listFiles(lessonId)`, `uploadFile(lessonId, file)`, `deleteFile(fileId)`, `getJobStatus(jobId)`; типы `SourceFile`, `ProcessingJob`
- [ ] T049 [P] [US3] Создать `frontend/src/lib/components/FileUpload.svelte` — drag-and-drop зона + кнопка выбора файла; отображает прогресс каждого файла отдельно (статус: В очереди / Обрабатывается X% / Готово ✓ / Ошибка ✗); polling `getJobStatus` каждые 3 с до завершения
- [ ] T050 [US3] Завершить `frontend/src/routes/lessons/[id]/+page.svelte` — секция загрузки файлов (`FileUpload.svelte`), список файлов с иконкой статуса, секция knowledge items (список карточек с вопросом/ответом/mastery); кнопка «Начать сессию»

**Checkpoint**: Загрузить PDF → обработка < 60 с → knowledge items видны в уроке; ошибка файла — отображается только для него.

---

## Phase 6: User Story 4 — Обучение через карточки (Priority: P2)

**Цель**: Пользователь запускает сессию с фильтрами, проходит карточки с оценкой 1–4, FSRS обновляет расписание.

**Independent Test**: Создать сессию mode=all для урока → пройти 5 карточек с оценками → карточки с оценкой 1 (Again) попадают в фильтр weak_areas в следующей сессии.

- [ ] T051 Реализовать `backend/src/services/fsrs_service.py` — обёртка над `py-fsrs`: `schedule_review(item, rating) → updated_item` (применяет FSRS v5, обновляет stability/difficulty/due_date/state/step); `compute_mastery_score(stability, last_review_date) → float` (формула R(t,S)); `get_due_items(session, filter_config) → list[KnowledgeItem]` (SQL-запрос с индексом `idx_ki_due_state`)
- [ ] T052 [US4] Реализовать `backend/src/api/sessions.py` — роутер: `POST /sessions` (создаёт LearningSession, вызывает fsrs_service.get_due_items, возвращает `{session, items}`); `POST /sessions/{id}/reviews` (записывает PerformanceRecord, вызывает schedule_review, обновляет KnowledgeItem, возвращает обновлённый item); `POST /sessions/{id}/finish` (устанавливает ended_at, возвращает итоги сессии)
- [ ] T053 [P] [US4] Создать `frontend/src/lib/api/sessions.ts` — TypeScript-клиент: `createSession(type, filterConfig)`, `submitReview(sessionId, itemId, rating)`, `finishSession(sessionId)`; типы `LearningSession`, `FilterConfig`, `KnowledgeItem`
- [ ] T054 [P] [US4] Создать `frontend/src/lib/components/FlashCard.svelte` — компонент карточки: лицевая сторона (вопрос), flip-анимация, оборотная сторона (ответ + контекст), кнопки рейтинга 1=Again/2=Hard/3=Good/4=Easy; клавиатурные шорткаты 1–4
- [ ] T055 [US4] Реализовать `frontend/src/routes/sessions/[id]/+page.svelte` — экран сессии: форма выбора уроков и режима (all/weak_areas/new_only/date_range), прогресс-бар, `FlashCard.svelte` для текущей карточки, экран результатов по завершении (% верных, count, кнопка «Повторить слабые»)

**Checkpoint**: Запустить сессию → пройти 10 карточек → увидеть экран результатов; слабые карточки (оценка 1) отображаются в фильтре weak_areas.

---

## Phase 7: User Story 5 — Дашборд прогресса (Priority: P2)

**Цель**: Пользователь видит общий прогресс, график сессий по дням, топ слабых элементов; фильтрация по workspace.

**Independent Test**: После 3 сессий открыть `/dashboard` → видны: total_items, avg_mastery_pct, график активности за неделю, список weak items (mastery < 70%); фильтр по workspace работает.

- [ ] T056 Реализовать `backend/src/services/dashboard_service.py` — агрегации: `get_stats(workspace_id=None) → DashboardStats` (total_items по state, due_today, avg_mastery_pct через mastery formula, lifetime_reviews, sessions_history за 30 дней с avg_accuracy); `get_weak_items(workspace_id=None) → list[KnowledgeItem]` (mastery < 70 OR difficulty > 7 OR lapse_count > 2); все запросы — через индексы из data-model.md
- [ ] T057 [US5] Реализовать `backend/src/api/dashboard.py` — роутер: `GET /dashboard/stats` (глобальная статистика), `GET /dashboard/workspaces/{id}/stats` (по workspace); оба → `DashboardStats`; вызывают dashboard_service
- [ ] T058 [P] [US5] Создать `frontend/src/lib/api/dashboard.ts` — TypeScript-клиент: `getDashboardStats()`, `getWorkspaceStats(workspaceId)`; тип `DashboardStats`
- [ ] T059 [P] [US5] Создать `frontend/src/lib/components/ProgressChart.svelte` — SVG-график сессий по дням (line chart); входные данные: `sessions_history[]`; отображает активность за 30 дней; минимальные зависимости (нет Chart.js — чистый SVG)
- [ ] T060 [US5] Создать `frontend/src/routes/dashboard/+page.svelte` — дашборд: карточки со статами (total/in_progress/mature/due_today/avg_mastery), `ProgressChart.svelte` с историей, список слабых элементов (сортировка по mastery asc), выпадающий фильтр по workspace

**Checkpoint**: Дашборд загружается < 2 с; фильтрация по workspace обновляет все метрики.

---

## Phase 8: User Story 6 — Чат-сессия с агентом (Priority: P3)

**Цель**: Пользователь ведёт свободный диалог с агентом в контексте урока; ответы стримятся через SSE.

**Independent Test**: Открыть чат с контекстом «Урока 1» → задать вопрос → получить ответ, опирающийся на контент урока; попросить игровую форму («угадай слово») → агент ведёт игру.

- [ ] T061 [P] [US6] Реализовать `backend/src/api/chat.py` — роутер: `POST /chat/sessions` (создаёт LearningSession type=chat, загружает контекст урока из KnowledgeItems); `POST /chat/sessions/{id}/messages` (SSE streaming через `StreamingResponse`; формирует system-промпт с контекстом урока; вызывает ai_client со streaming; каждый чанк → `data: {text}\n\n`)
- [ ] T062 [US6] Создать `frontend/src/routes/chat/+page.svelte` — интерфейс чата: выбор уроков для контекста, textarea для ввода, история сообщений (user/assistant), SSE-получение ответа через `EventSource`, индикатор набора текста

**Checkpoint**: Чат отвечает на вопрос по материалу урока; ответ появляется постепенно (streaming).

---

## Phase 9: Polish & Cross-Cutting Concerns

**Цель**: Интеграция всех компонентов, hardening, производительность.

- [ ] T063 [P] Дополнить `docker-compose.yml`: healthchecks для backend (`/api/v1/health`), worker и redis; `depends_on` с condition: service_healthy; named volumes `sqlite_data`, `file_storage`, `redis_data`; добавить `GET /api/v1/health` эндпоинт в `backend/src/main.py`
- [ ] T064 [P] Реализовать каскадные удаления в `backend/src/api/`: при `DELETE /files/{id}` — физическое удаление файла с ФС через file_utils.py; при `DELETE /workspaces/{id}` — удаление всех дочерних записей + файлов; при `DELETE /lessons/{id}` — то же; добавить `DELETE /workspaces/{id}?confirm=true` guard
- [ ] T065 [P] Настроить `backend/src/config.py`: enforcement лимита MAX_FILE_SIZE_BYTES в `api/files.py`; fallback WHISPER_MODEL=small при OOM; CORS_ORIGINS из env; добавить `X-Request-ID` middleware в main.py
- [ ] T066 Проверить все роутеры на соответствие `ErrorResponse {error, message, details}`: HTTPException → кастомный handler; непредвиденные исключения → 500 с `error="internal_error"`; убедиться, что ни один эндпоинт не возвращает plain text при ошибке
- [ ] T067 Пройти сквозной сценарий quickstart.md: `docker compose up --build` → открыть браузер → подключить Anthropic → создать workspace → загрузить PDF → дождаться обработки → начать карточную сессию → открыть дашборд; зафиксировать результат в `specs/001-ai-study-platform/quickstart-validation.md`

**Checkpoint**: Все User Stories работают end-to-end; quickstart.md воспроизводится за < 5 мин.

---

## Dependencies & Execution Order

### Phase Dependencies

```
Phase 1: Setup              → нет зависимостей, старт немедленно
Phase 2: Foundational       → зависит от Phase 1; БЛОКИРУЕТ все Phase 3+
Phase 3: US1 Agent          → зависит от Phase 2
Phase 4: US2 Workspaces     → зависит от Phase 2 (независима от US1)
Phase 5: US3 File Upload    → зависит от Phase 2 + US1 (нужен ai_client) + US2 (нужен lesson)
Phase 6: US4 Flashcards     → зависит от Phase 2 + US3 (нужны knowledge items)
Phase 7: US5 Dashboard      → зависит от Phase 2 + US4 (нужна история сессий)
Phase 8: US6 Chat           → зависит от Phase 2 + US1 (ai_client) + US3 (контекст)
Phase 9: Polish             → зависит от всех Phase 3–8
```

### User Story Dependencies

```
US1 (Agent)     → после Phase 2. Нет зависимостей от других US.
US2 (Workspace) → после Phase 2. Нет зависимостей от других US.
US3 (Files)     → требует US1 (ai_client для item_extractor) + US2 (lesson_id для загрузки).
US4 (Sessions)  → требует US3 (knowledge items должны существовать).
US5 (Dashboard) → требует US4 (данные сессий для графиков).
US6 (Chat)      → требует US1 (ai_client) + US3 (контекст урока из knowledge items).
```

### Внутри User Story

- Utilities/Config → Database → Models → Migration → App entry
- Services → Routers → Frontend API client → Frontend components → Frontend pages

### Parallel Opportunities

**Phase 2** (после T010 — database.py готов):
- T011–T018 (все модели) — полностью параллельно, разные файлы

**Phase 3** (US1):
- T024 (ai_client.py) параллельно с T023 (oauth_service.py) — оба зависят только от T022 (auth_store)
- T026 (AgentStatusBadge) и T027 (api/agent.ts) — параллельно, разные файлы

**Phase 4** (US2):
- T030 (workspaces.py) + T031 (lessons.py) + T032 (workspaces.ts) + T033 (lessons.ts) — все параллельно

**Phase 5** (US3):
- T038–T042 (все процессоры) — полностью параллельно, разные файлы
- T048 (files.ts) + T049 (FileUpload.svelte) — параллельно

---

## Parallel Example: Phase 2

```bash
# После T010 (database.py) — запускаем все модели одновременно:
Task: "backend/src/models/workspace.py"         # T011
Task: "backend/src/models/lesson.py"            # T012
Task: "backend/src/models/agent_config.py"      # T013
Task: "backend/src/models/source_file.py"       # T014
Task: "backend/src/models/knowledge_item.py"    # T015
Task: "backend/src/models/learning_session.py"  # T016
Task: "backend/src/models/performance_record.py"# T017
Task: "backend/src/models/processing_job.py"    # T018
# → Затем T019 (Migration) + T020 (main.py) параллельно
```

## Parallel Example: Phase 5 (US3 — Processors)

```bash
# После T037 (text_chunker.py) — все процессоры параллельно:
Task: "backend/src/processing/pdf_processor.py"   # T038
Task: "backend/src/processing/pptx_processor.py"  # T039
Task: "backend/src/processing/docx_processor.py"  # T040
Task: "backend/src/processing/audio_processor.py" # T041
Task: "backend/src/processing/image_processor.py" # T042
# → Затем T043 (item_extractor) → T044 (pipeline) → T045 (worker)
```

---

## Implementation Strategy

### MVP First (User Stories 1–3)

1. Завершить **Phase 1** (Setup)
2. Завершить **Phase 2** (Foundational) — критический блокер
3. Завершить **Phase 3** (US1 — Agent) — без агента невозможна обработка
4. Завершить **Phase 4** (US2 — Workspace/Lessons) — структура данных
5. Завершить **Phase 5** (US3 — Files) — ключевая ценность платформы
6. **СТОП и ПРОВЕРКА**: загрузить PDF → увидеть knowledge items → подтвердить MVP
7. Деплой/демо если готово

### Incremental Delivery

```
Phase 1+2 (Setup+Foundation) → каркас готов, запускается
Phase 3 (US1)                → агент подключён, статус виден в шапке
Phase 4 (US2)                → workspace/lesson CRUD работает
Phase 5 (US3)                → файлы обрабатываются, items появляются  ← MVP!
Phase 6 (US4)                → карточные сессии работают
Phase 7 (US5)                → дашборд показывает прогресс
Phase 8 (US6)                → чат с агентом
Phase 9 (Polish)             → production-ready
```

### Parallel Team Strategy

После завершения Phase 2 (Foundational):
- **Developer A**: US1 (Agent connection) → US3 depends on this
- **Developer B**: US2 (Workspace/Lessons)
- **Developer C**: US5/US4 (Dashboard/Sessions) — после US3
- **Developer D**: US6 (Chat) — после US1

---

## Notes

- [P] задачи = разные файлы, нет незавершённых зависимостей внутри фазы
- [Story] метка связывает задачу с конкретной User Story для трассируемости
- Каждая User Story независимо тестируема на своём checkpoint
- Коммит после каждой задачи или логической группы
- Проверяй checkpoint перед переходом к следующей фазе
- Хранение учётных данных — только `~/.config/study-kit/credentials.json.enc`, никогда в SQLite или Docker volume
- Все API роутеры — тонкий слой (только HTTP-адаптер); бизнес-логика — в services/
