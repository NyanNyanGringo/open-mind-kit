# Implementation Plan: AI Study Platform — Core Application

**Branch**: `001-ai-study-platform` | **Date**: 2026-02-18 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `specs/001-ai-study-platform/spec.md`

---

## Summary

Локальное веб-приложение для обучения: пользователь загружает файлы любых форматов
(PDF, PPTX, DOCX, аудио, видео, изображения) в уроки рабочего пространства; AI-агент
(Anthropic/OpenAI) асинхронно обрабатывает файлы и извлекает knowledge items; пользователь
практикуется через FSRS-планированные карточки с дашбордом прогресса.

Стек: Python 3.12/FastAPI + SvelteKit 2/Svelte 5 + SQLite WAL + ARQ/Redis,
деплой через Docker Compose, полностью локально.

---

## Technical Context

**Language/Version**: Python 3.12 (backend), Node.js 22 LTS (frontend build)
**Primary Dependencies**:
- Backend: FastAPI 0.129, SQLModel 0.0.34, aiosqlite 0.22, ARQ 0.26, authlib 1.3,
  py-fsrs, PyMuPDF 1.24, faster-whisper 1.0.3, python-pptx 1.0, mammoth 1.7,
  easyocr 1.7.2, langchain-text-splitters 0.3, cryptography 43, portalocker 2.8
- Frontend: SvelteKit 2.52, Svelte 5.51, TypeScript 5.9, Vite 7.3

**Storage**: SQLite WAL (via SQLModel + aiosqlite) + локальная ФС для тел файлов
**Testing**: pytest + pytest-asyncio (backend); vitest + playwright (frontend)
**Target Platform**: Docker Compose на Linux/macOS/Windows desktop (single-user)
**Project Type**: web — backend + frontend в отдельных контейнерах
**Performance Goals**:
- PDF до 50 МБ: обработан за < 60 с (SC-002)
- Аудио/видео до 30 мин: транскрипция < 5 мин (SC-003)
- Карточная сессия: запуск мгновенный (SC-004)
- Дашборд: загрузка < 2 с при 1 000 сессий (SC-006)

**Constraints**:
- Полностью оффлайн после первоначальной настройки (FR-012)
- Данные не покидают машину пользователя без явного согласия
- p95 API-ответов < 500 мс для read-операций

**Scale/Scope**: Один пользователь; неограниченная история; 10 000+ knowledge items

---

## Constitution Check

*GATE: проверено до начала Phase 0. Повторная проверка после Phase 1.*

### I. Security / Privacy ✅

- Учётные данные хранятся исключительно в `~/.config/study-kit/credentials.json.enc`
  (Fernet AES-128-CBC, PBKDF2 1 200 000 итераций — OWASP 2025)
- Данные пользователя не покидают машину без явного согласия (FR-012)
- OAuth-поток: явная авторизация в браузере, PKCE защита от CSRF
- Принцип наименьших привилегий: ARQ worker и FastAPI не имеют доступа к ФС за пределами `/data/`

### II. Performance ✅

- ARQ worker — неблокирующая обработка файлов (возвращает `job_id` немедленно)
- SQLite WAL: параллельные чтения не блокируют writer'а
- Все list-эндпоинты пагинированы (см. `PaginatedKnowledgeItems` в contracts)
- Индексы: `(lesson_id, due_date)`, `(due_date, state)`, `(knowledge_item_id, reviewed_at DESC)` — см. data-model.md
- p95 targets закреплены в SC-001..SC-007

### III. Scalability ✅

- Worker — отдельный Docker-контейнер, масштабируется независимо от API
- Вычисления (ARQ worker) отделены от хранения (SQLite + ФС)
- SQLite WAL при 10× росте данных (100 000 items) даёт < 100 мс запроса с индексами
- При росте до multi-user: SQLite заменяется PostgreSQL (миграция Alembic без изменений бизнес-логики)

### IV. Backward Compatibility ✅

- Alembic управляет схемой: каждое изменение — миграция с upgrade/downgrade
- API версионирован под `/api/v1`: ломающие изменения → `/api/v2` + deprecation notice
- Docker Compose volumes сохраняются при `docker compose down` (данные не теряются)

### V. Explicitness ✅

- Статус обработки каждого файла всегда виден в UI (FR-013)
- Статус агента (`connected / expired / error`) отображается в header
- ARQ job status polling явный: фронт запрашивает `GET /api/v1/jobs/{id}` — нет скрытых обновлений
- Все фоновые процессы инициированы явным действием пользователя (загрузка файла)

### VI. UX Consistency ✅

- Единый формат ошибок: `ErrorResponse {error, message, details}` из всех эндпоинтов
- Состояния загрузки и ошибки стандартизированы в SvelteKit layout-компонентах
- Рейтинг карточек: один и тот же виджет 1–4 используется во всех типах сессий

### VII. Testability ✅

- FastAPI dependency injection: все внешние зависимости (DB, ARQ, AI client) инджектируются → легко мокировать
- Services layer изолирован от API layer (нет прямых вызовов БД в роутерах)
- FSRS-логика в `py-fsrs` — pure functions, тестируемы без инфраструктуры
- ARQ задачи — `async def`, тестируются с `pytest-asyncio` без реального Redis
- Критические пути (OAuth flow, file processing) покрыты интеграционными тестами

---

## Project Structure

### Documentation (this feature)

```text
specs/001-ai-study-platform/
├── plan.md              # Этот файл
├── research.md          # Phase 0: все unknowns разрешены
├── data-model.md        # Phase 1: схема БД + индексы
├── quickstart.md        # Phase 1: как запустить и использовать
├── contracts/
│   └── openapi.yaml     # Phase 1: REST API контракт
└── tasks.md             # Phase 2: задачи (/speckit.tasks — не создаётся здесь)
```

### Source Code (repository root)

```text
backend/
├── src/
│   ├── main.py                  # FastAPI app entry point
│   ├── config.py                # Все настройки из env (константы, лимиты, пути)
│   ├── database.py              # SQLite-подключение, SQLModel setup, PRAGMA
│   ├── models/                  # SQLModel table definitions
│   │   ├── workspace.py
│   │   ├── lesson.py
│   │   ├── source_file.py
│   │   ├── knowledge_item.py
│   │   ├── learning_session.py
│   │   ├── performance_record.py
│   │   ├── agent_config.py
│   │   └── processing_job.py
│   ├── api/                     # FastAPI routers (тонкий слой: только HTTP-адаптер)
│   │   ├── agent.py             # /agent/status, /agent/connect/*
│   │   ├── workspaces.py        # /workspaces
│   │   ├── lessons.py           # /lessons
│   │   ├── files.py             # /lessons/{id}/files, /files/{id}
│   │   ├── items.py             # /items
│   │   ├── sessions.py          # /sessions
│   │   ├── dashboard.py         # /dashboard
│   │   └── chat.py              # /chat
│   ├── services/                # Вся бизнес-логика
│   │   ├── oauth_service.py     # PKCE flow, callback server, token refresh
│   │   ├── auth_store.py        # Fernet-шифрование, portalocker
│   │   ├── ai_client.py         # Единый клиент для Anthropic + OpenAI
│   │   ├── fsrs_service.py      # Обёртка py-fsrs: обновление карточки, выборка очереди
│   │   └── dashboard_service.py # Агрегации для дашборда
│   ├── processing/              # File processing pipeline
│   │   ├── worker.py            # ARQ worker: регистрация задач, настройки retry
│   │   ├── pipeline.py          # Оркестратор: dispatch по MIME-типу
│   │   ├── pdf_processor.py     # PyMuPDF + pdfplumber
│   │   ├── pptx_processor.py    # python-pptx
│   │   ├── docx_processor.py    # mammoth
│   │   ├── audio_processor.py   # faster-whisper + ffmpeg-python
│   │   ├── image_processor.py   # easyocr
│   │   ├── text_chunker.py      # RecursiveCharacterTextSplitter
│   │   └── item_extractor.py    # AI: извлечение knowledge items из чанков
│   └── utilities/               # Общий код (нет дублирования между модулями)
│       ├── file_utils.py        # MIME-detection, path management, checksum
│       ├── crypto_utils.py      # Fernet helpers, PBKDF2 key derivation
│       └── error_utils.py       # Стандартизация ошибок, структурированный logging
├── migrations/                  # Alembic migrations
├── tests/
│   ├── unit/                    # Pure function тесты (fsrs, chunker, processors)
│   ├── integration/             # API + DB тесты (pytest + httpx TestClient)
│   └── contract/                # Тесты соответствия OpenAPI-контракту
├── Dockerfile
└── pyproject.toml

frontend/
├── src/
│   ├── lib/
│   │   ├── components/          # Переиспользуемые Svelte-компоненты
│   │   │   ├── FileUpload.svelte
│   │   │   ├── FlashCard.svelte
│   │   │   ├── ProgressChart.svelte
│   │   │   ├── AgentStatusBadge.svelte
│   │   │   └── ErrorMessage.svelte
│   │   └── api/                 # TypeScript-клиент к backend API
│   │       ├── workspaces.ts
│   │       ├── lessons.ts
│   │       ├── files.ts
│   │       ├── sessions.ts
│   │       └── dashboard.ts
│   └── routes/                  # SvelteKit file-based routing
│       ├── +layout.svelte       # Глобальный layout с AgentStatusBadge
│       ├── +page.svelte         # Главный экран (список пространств + дашборд)
│       ├── settings/+page.svelte          # Подключение агентов
│       ├── workspaces/[id]/+page.svelte   # Список уроков
│       ├── lessons/[id]/+page.svelte      # Список файлов + knowledge items
│       └── sessions/[id]/+page.svelte     # Карточная сессия
├── package.json
└── Dockerfile

docker-compose.yml              # Оркестрация: frontend, backend, worker, redis
.env.example                    # Шаблон переменных окружения
```

**Structure Decision**: Option 2 — Web application (backend + frontend в отдельных контейнерах).
Выбрано потому что:
1. Весь стек обработки файлов — Python; фронтенд — Node.js (SvelteKit SSR)
2. Разделение позволяет масштабировать worker независимо от API
3. Стандартный паттерн для FastAPI + SvelteKit + Docker Compose

---

## Complexity Tracking

> Нет нарушений конституционных принципов — таблица пустая.

---

## Phase 0: Research Summary

Все unknowns разрешены. Подробности — в [research.md](./research.md).

| Решение | Выбор |
|---------|-------|
| Backend | Python 3.12 + FastAPI 0.129 |
| Frontend | SvelteKit 2 + Svelte 5 + TypeScript |
| Database | SQLite WAL + SQLModel + aiosqlite |
| Task queue | ARQ 0.26 + Redis 7 |
| OAuth (OpenAI) | PKCE + локальный callback-сервер (authlib) |
| Auth (Anthropic) | API-key paste-flow (OAuth2 не поддерживается) |
| Token storage | Fernet + PBKDF2 + portalocker |
| PDF | PyMuPDF + pdfplumber (таблицы) |
| Audio/Video | faster-whisper + large-v3-turbo |
| OCR | easyocr + pytesseract (чистые сканы) |
| Chunking | RecursiveCharacterTextSplitter, 512 токенов, 64 overlap |
| Spaced repetition | FSRS v5 + py-fsrs |
| Mastery score | R(t,S) = (1 + 19/81 * t/S)^(-0.5) * 100% |

---

## Phase 1: Design Artifacts

- **[data-model.md](./data-model.md)** — все сущности, поля, индексы, state-диаграммы
- **[contracts/openapi.yaml](./contracts/openapi.yaml)** — полный REST API контракт
- **[quickstart.md](./quickstart.md)** — развёртывание и первый сценарий использования

### Re-check Constitution Check (post-design)

После проектирования API и схемы данных:

- **I. Security**: ✅ `agent_configs` не хранит credentials; Fernet-файл вне Docker-volume
- **II. Performance**: ✅ Все сущности имеют план индексации (data-model.md); пагинация во всех list-эндпоинтах
- **III. Scalability**: ✅ Worker — отдельный сервис; вычисления отделены от хранения
- **IV. Backward Compatibility**: ✅ API под `/api/v1`; Alembic migrations
- **V. Explicitness**: ✅ Все статусы явно отражены в API и DB (processing_status, agent status)
- **VI. UX Consistency**: ✅ `ErrorResponse` — единый формат; пагинация — единая схема
- **VII. Testability**: ✅ Services layer изолирован; FSRS — pure functions; ARQ tasks тестируемы

**Результат**: Все 7 принципов соблюдены. Нарушений нет. Переходим к /speckit.tasks.
