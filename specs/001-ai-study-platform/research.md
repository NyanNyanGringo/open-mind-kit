# Research: AI Study Platform

**Branch**: `001-ai-study-platform` | **Date**: 2026-02-18
**Phase**: 0 — Resolve all technical unknowns before design

---

## 1. Backend Framework

**Decision**: FastAPI 0.129 (Python 3.12)

**Rationale**: Весь стек обработки файлов (PDF, PPTX, DOCX, Whisper, OCR) существует
как Python-экосистема. Go и Node.js потребовали бы shell-вызовов или FFI к Python-инструментам,
что нивелирует их преимущества. FastAPI — async-native (Starlette + anyio), автогенерирует
OpenAPI-документацию, имеет первоклассную поддержку Pydantic v2 для валидации, используется
Uber и Netflix для ML-сервисов.

**Alternatives considered**:
- Go — отклонён: нет зрелых библиотек для PDF/PPTX/DOCX/Whisper
- Node.js/Fastify — отклонён: все AI/ML-библиотеки Python-first

**Key versions**: `fastapi==0.129`, `uvicorn[standard]==0.41`, `pydantic==2.12`, Python 3.12 LTS

---

## 2. Frontend Framework

**Decision**: SvelteKit 2 + Svelte 5 + TypeScript

**Rationale**: Скомпилированный фреймворк без virtual DOM — минимальный bundle для
однопользовательского локального приложения. Svelte 5 runes (`$state`, `$derived`, `$effect`)
устраняют потребность в отдельных state-менеджерах. Наивысший developer satisfaction в
Stack Overflow 2024 (72.8%). SvelteKit 2 включает маршрутизацию и SSR из коробки.
`@sveltejs/adapter-node` позволяет запускать как Node.js-сервер внутри Docker.

**Alternatives considered**:
- React/Next.js — отклонён: virtual DOM overhead, сложный state management для single-user app
- Vue 3 — отклонён: нет значимых преимуществ перед SvelteKit для данного use case

**Key versions**: `svelte@5.51`, `@sveltejs/kit@2.52`, `vite@7.3`, `typescript@5.9`

---

## 3. Database

**Decision**: SQLite с WAL-режимом (через SQLModel + aiosqlite)

**Rationale**: PostgreSQL в Docker Compose для однопользовательского приложения — избыточный
overhead (отдельный контейнер, connection pooling, 200+ MB образ). SQLite с WAL устраняет
классическую проблему конкурентности: множество читателей не блокируют друг друга и writer'а.
SQLite на 35% быстрее файловой системы для малых BLOB (<50 KB). Вся база — один файл:
тривиальный backup через `cp`. Alembic обеспечивает миграции схемы.

Конфигурация при подключении:
```sql
PRAGMA journal_mode=WAL;
PRAGMA synchronous=NORMAL;
PRAGMA foreign_keys=ON;
PRAGMA busy_timeout=5000;
```

**Хранение файлов**: тела файлов >50 KB на локальной ФС (`/data/uploads/`),
метаданные — в SQLite.

**Alternatives considered**:
- PostgreSQL — отклонён: нет преимуществ для single-user, требует отдельного контейнера
- SQLite без WAL — отклонён: блокировки при параллельных записях из ARQ worker'а

**Key versions**: `sqlmodel==0.0.34`, `aiosqlite==0.22`, `alembic==1.18`

---

## 4. Task Queue

**Decision**: ARQ + Redis 7

**Rationale**: FastAPI BackgroundTasks исключён — нет status tracking, задачи теряются при
перезапуске. Celery отклонён — не async-native (обходные пути с gevent), избыточен для
однопользовательского deployment. ARQ — async-native (`async def` задачи), job status хранится
в Redis, поддерживает retry/timeout/defer. Сообщество 2025 называет ARQ "recommended choice
для asyncio-native Python services".

**Архитектурный поток**:
1. FastAPI принимает файл → сохраняет на ФС → создаёт DB-запись → enqueue ARQ job → возвращает `{job_id}`
2. ARQ worker (отдельный контейнер) обрабатывает файл
3. Frontend polling `GET /api/jobs/{job_id}` или SSE для realtime-обновлений

**Alternatives considered**:
- Celery — отклонён: не async-native, overkill для single-worker
- FastAPI BackgroundTasks — отклонён: нет статуса, задачи не персистентны
- SAQ — валидная альтернатива, но меньше community + production-референсов

**Key versions**: `arq==0.26`, `redis==5.x` (Redis 7-alpine в Docker)

---

## 5. OAuth2 / AI Agent Connection

**Decision**: PKCE OAuth2 для OpenAI/Codex; API-key paste-flow для Anthropic

**Rationale**: Anthropic API **не поддерживает OAuth2** — только API-ключи через заголовок
`x-api-key`. Paste-flow аналогичен `claude setup-token` в OpenClaw: пользователь генерирует
ключ на `platform.anthropic.com`, вставляет в UI. OpenAI поддерживает полный PKCE OAuth2-поток.

**PKCE flow** (для OpenAI):
1. Генерация `code_verifier` (64 random bytes → base64url) + `code_challenge` (SHA-256)
2. Локальный callback-сервер на `http://127.0.0.1:{random_port}/auth/callback`
3. Открытие браузера на authorization endpoint
4. Перехват `?code=...&state=...` через однократный HTTP-обработчик
5. Обмен `code` + `code_verifier` на токены

**Хранение**: `cryptography` Fernet (AES-128-CBC + HMAC-SHA256) с PBKDF2-производным ключом
(1 200 000 итераций — OWASP 2025). JSON-файл организован по `agentId`.
Файловые блокировки: `portalocker` (cross-platform, context manager, timeout).

**Libraries**: `authlib>=1.3`, `httpx>=0.27`, `cryptography>=43`, `portalocker>=2.8`

---

## 6. File Processing Pipeline

### PDF
**Decision**: PyMuPDF (fitz) основной, pdfplumber для таблиц

Бенчмарк (7 031 страница): PyMuPDF — fastest (8 с), 96% точность, встроенный OCR через
Tesseract. pdfplumber — slowest (35× медленнее), но единственный с надёжной spatial-экстракцией таблиц.

**License note**: PyMuPDF — AGPL-3.0. Если проект требует MIT-совместимости, использовать
`pypdf>=4.3` (MIT, 96% точность, медленнее).

**Libraries**: `PyMuPDF>=1.24`, `pdfplumber>=0.11` (для таблиц)

### PPTX / PPT
**Decision**: python-pptx 1.0.0 (единственная Python-библиотека для PPTX)

Для `.ppt` (legacy): preprocessing через LibreOffice CLI (`soffice --headless --convert-to pptx`).

### DOCX
**Decision**: `mammoth>=1.7` для AI-инджестинга (конвертирует в чистый Markdown),
`python-docx>=1.1` для низкоуровневого доступа к структуре.

### Audio / Video
**Decision**: `faster-whisper>=1.0.3` с моделью `large-v3-turbo`

Сравнение: faster-whisper на 4× быстрее openai-whisper при одинаковой точности;
`large-v3-turbo` — на 6× быстрее large-v3 при WER 7.75% (+0.35%). CPU-совместим,
поддерживает Apple Silicon через CoreML, INT8-квантизация снижает требования к памяти.

Извлечение аудио из видео: `ffmpeg-python>=0.2` (wrapper над системным ffmpeg).

### OCR для изображений
**Decision**: `easyocr>=1.7.2` основной (deep-learning, устойчив к шуму/наклону);
`pytesseract>=0.3.10` для чистых отсканированных документов; Claude Vision API как fallback
для сложных макетов (форм, графиков).

### Chunking
**Decision**: `RecursiveCharacterTextSplitter` (langchain-text-splitters), 512 токенов, 64 overlap

Бенчмарки 2025: semantic chunking +70% точность vs fixed-size, но в 3× сложнее. Recursive —
оптимальный баланс. Специфика по форматам:
- PDF / transcript: 512 токенов, 64 overlap
- PPTX: граница слайда = граница чанка
- DOCX (Markdown): сплит по heading-разделителям (`## `, `# `), затем recursive
- Audio: 512 токенов, 128 overlap (речь лишена жёстких границ предложений)

**Libraries**: `langchain-text-splitters>=0.3`, `tiktoken>=0.7`

---

## 7. Spaced Repetition Algorithm

**Decision**: FSRS v5 с default weights (py-fsrs)

**Rationale**: SM-2 — устаревший алгоритм 1987 года с "ease hell" (один неверный ответ
необратимо деградирует карточку). FSRS v5 — алгоритм на основе DSR-модели (Difficulty,
Stability, Retrievability), обученный на миллионах анки-отзывов, с 2023 — дефолт в Anki.
Default weight-вектор W (21 значение) опубликован и работает без cold-start.
Retrievability R(t, S) даёт естественную метрику "mastery score" (0–100%).

**Rating scale**: 1=Again, 2=Hard, 3=Good, 4=Easy (вместо SM-2 0-5)

**Mastery score formula**:
```
R(t, S) = (1 + (19/81) * (t / S)) ^ (-0.5) * 100%
```
где `t` = дней с последнего повторения, `S` = stability (дней до R < 90%)

**Weak area threshold**: `mastery_score < 70` OR `difficulty > 7` OR `lapse_count > 2`

**Implementation**: `py-fsrs` (open-spaced-repetition/py-fsrs, MIT license)

---

## 8. Docker Compose Architecture

```
Services:
  frontend  (Node.js 22)    — SvelteKit SSR, port 3000
  backend   (Python 3.12)   — FastAPI + Uvicorn, port 8000
  worker    (Python 3.12)   — ARQ worker (same image as backend)
  redis     (Redis 7-alpine) — ARQ broker, port 6379

Volumes:
  sqlite_data   → /data/app.db
  file_storage  → /data/uploads/, /data/processed/
  redis_data    → Redis persistence
```

---

## Resolved Unknowns Summary

| Unknown | Decision | Source |
|---------|----------|--------|
| Backend language | Python 3.12 + FastAPI | Research agent 1 |
| Frontend | SvelteKit 2 + Svelte 5 | Research agent 1 |
| Database | SQLite WAL + SQLModel | Research agent 1 |
| Task queue | ARQ + Redis | Research agent 1 |
| Anthropic OAuth | API key paste-flow (no OAuth2) | Research agent 2 |
| OpenAI OAuth | PKCE + local callback server | Research agent 2 |
| Token storage | Fernet + PBKDF2 + portalocker | Research agent 2 |
| PDF extraction | PyMuPDF (primary) + pdfplumber (tables) | Research agent 2 |
| Audio transcription | faster-whisper + large-v3-turbo | Research agent 2 |
| Spaced repetition | FSRS v5 + py-fsrs | Research agent 3 |
| Mastery score | R(t,S) power-law formula | Research agent 3 |
