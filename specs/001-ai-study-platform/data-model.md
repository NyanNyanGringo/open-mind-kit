# Data Model: AI Study Platform

**Branch**: `001-ai-study-platform` | **Date**: 2026-02-18
**Algorithm**: FSRS v5 (py-fsrs) | **DB**: SQLite WAL

---

## Entity Overview

```
Workspace (1) ──< Lesson (1) ──< SourceFile
                               └──< KnowledgeItem ──< PerformanceRecord
                                         │
                               LearningSession >──< PerformanceRecord
AgentConfig (singleton per provider)
ProcessingJob (1:1 with SourceFile)
```

---

## Table: `workspaces`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | Идентификатор |
| `name` | TEXT | NOT NULL | Название (макс. 200 символов) |
| `description` | TEXT | nullable | Опциональное описание |
| `created_at` | DATETIME | NOT NULL, default now() | Дата создания |
| `updated_at` | DATETIME | NOT NULL, default now() | Дата последнего изменения |

**Indexes**: нет (таблица маленькая, full-scan допустим)

**Validation**:
- `name` не может быть пустым или состоять только из пробелов
- `name` уникален (case-insensitive)

---

## Table: `lessons`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | Идентификатор |
| `workspace_id` | UUID | FK → workspaces, NOT NULL | Родительское пространство |
| `name` | TEXT | NOT NULL | Название урока |
| `order_index` | INTEGER | NOT NULL, default 0 | Порядок в списке (0-based) |
| `created_at` | DATETIME | NOT NULL, default now() | |
| `updated_at` | DATETIME | NOT NULL, default now() | |

**Indexes**:
- `idx_lessons_workspace` ON `(workspace_id, order_index)` — список уроков в рабочем пространстве

**Validation**:
- `name` не пустое
- `order_index >= 0`
- При удалении Workspace — каскадное удаление Lessons

---

## Table: `source_files`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | |
| `lesson_id` | UUID | FK → lessons, NOT NULL | |
| `original_name` | TEXT | NOT NULL | Исходное имя файла (включая расширение) |
| `mime_type` | TEXT | NOT NULL | MIME-тип (определяется при загрузке) |
| `size_bytes` | INTEGER | NOT NULL | Размер в байтах |
| `file_path` | TEXT | NOT NULL | Относительный путь в `/data/uploads/` |
| `processing_status` | TEXT | NOT NULL, default 'queued' | `queued` / `processing` / `completed` / `failed` |
| `error_message` | TEXT | nullable | Сообщение об ошибке если status = 'failed' |
| `uploaded_at` | DATETIME | NOT NULL, default now() | |
| `processed_at` | DATETIME | nullable | Дата завершения обработки |

**Indexes**:
- `idx_source_files_lesson` ON `(lesson_id, processing_status)` — список файлов урока
- `idx_source_files_status` ON `(processing_status)` — мониторинг очереди

**Validation**:
- `processing_status` — только значения из enum
- `size_bytes > 0`
- Допустимые `mime_type` ограничены белым списком (PDF, PPTX, DOCX, аудио, видео, изображения, текст)
- При удалении файла из БД — удалить физический файл с диска

**File path convention**: `/data/uploads/{lesson_id}/{file_id}/{original_name}`

---

## Table: `knowledge_items`

Основная таблица. Хранит FSRS-состояние каждой карточки.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | |
| `source_file_id` | UUID | FK → source_files, nullable | Источник (nullable: элемент может быть создан вручную) |
| `lesson_id` | UUID | FK → lessons, NOT NULL | Денормализовано для быстрых запросов |
| `question` | TEXT | NOT NULL | Лицевая сторона карточки |
| `answer` | TEXT | NOT NULL | Оборотная сторона карточки |
| `context` | TEXT | nullable | Выдержка из источника (контекст для агента) |
| **FSRS поля** | | | |
| `stability` | REAL | nullable | Дней до R < 90%; NULL = ещё не повторялась |
| `difficulty` | REAL | nullable | Сложность [1, 10]; NULL = не повторялась |
| `due_date` | DATE | NOT NULL, default today | Дата следующего повторения |
| `last_review_date` | DATETIME | nullable | Для вычисления t = дней с последнего повтора |
| `state` | TEXT | NOT NULL, default 'new' | `new` / `learning` / `review` / `relearning` |
| `step` | INTEGER | NOT NULL, default 0 | Позиция в learning/relearning шагах |
| **Агрегаты** (денормализованы для dashboard) | | | |
| `total_reviews` | INTEGER | NOT NULL, default 0 | Всего повторений |
| `correct_reviews` | INTEGER | NOT NULL, default 0 | Верных повторений |
| `lapse_count` | INTEGER | NOT NULL, default 0 | Упавших зрелых карточек |
| `streak` | INTEGER | NOT NULL, default 0 | Текущая серия верных ответов |
| `created_at` | DATETIME | NOT NULL, default now() | |
| `updated_at` | DATETIME | NOT NULL, default now() | |

**Indexes**:
- `idx_ki_lesson_due` ON `(lesson_id, due_date)` — основной путь: карточки урока на сегодня
- `idx_ki_lesson_state` ON `(lesson_id, state)` — фильтрация по статусу (new/learning/review)
- `idx_ki_due_state` ON `(due_date, state)` — глобальная очередь повторений

**Mastery score** (вычисляется в runtime, не хранится):
```python
def compute_mastery_score(stability: float | None, last_review_date: datetime | None) -> float:
    """Возвращает 0–100. Значение убывает между сессиями (память угасает)."""
    if stability is None or last_review_date is None:
        return 0.0
    elapsed_days = (datetime.utcnow() - last_review_date).total_seconds() / 86400
    return pow(1 + (19 / 81) * (elapsed_days / stability), -0.5) * 100
```

**Weak area criteria**:
```sql
WHERE mastery_score < 70          -- низкая восстановимость
   OR difficulty > 7.0             -- высокая сложность
   OR (lapse_count > 2 AND total_reviews > 5)  -- повторные провалы
```

---

## Table: `learning_sessions`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | |
| `session_type` | TEXT | NOT NULL | `flashcards` / `test` / `chat` |
| `filter_config` | TEXT | NOT NULL | JSON: `{lesson_ids, mode, date_from, date_to}` |
| `started_at` | DATETIME | NOT NULL, default now() | |
| `ended_at` | DATETIME | nullable | NULL = сессия активна |
| `total_items` | INTEGER | NOT NULL, default 0 | Всего карточек в сессии |
| `correct_items` | INTEGER | NOT NULL, default 0 | Верных ответов |

**`filter_config` JSON schema**:
```json
{
  "lesson_ids": ["uuid1", "uuid2"],
  "mode": "all" | "weak_areas" | "date_range" | "new_only",
  "date_from": "2026-01-01",
  "date_to": "2026-02-18"
}
```

**Indexes**:
- `idx_sessions_started` ON `(started_at DESC)` — история сессий (dashboard)

---

## Table: `performance_records`

Append-only лог. Одна строка = один ответ на карточку.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | |
| `knowledge_item_id` | UUID | FK → knowledge_items, NOT NULL | |
| `session_id` | UUID | FK → learning_sessions, NOT NULL | |
| `reviewed_at` | DATETIME | NOT NULL, default now() | |
| `rating` | INTEGER | NOT NULL | 1=Again, 2=Hard, 3=Good, 4=Easy |
| `was_correct` | BOOLEAN | NOT NULL | `rating >= 2` |
| `stability_before` | REAL | nullable | FSRS: stability до ответа |
| `difficulty_before` | REAL | nullable | FSRS: difficulty до ответа |
| `stability_after` | REAL | nullable | FSRS: stability после ответа |
| `difficulty_after` | REAL | nullable | FSRS: difficulty после ответа |
| `scheduled_interval_days` | INTEGER | nullable | Запланированный интервал в днях |
| `elapsed_days` | INTEGER | nullable | Дней прошло с последнего повтора |

**Indexes**:
- `idx_perf_item` ON `(knowledge_item_id, reviewed_at DESC)` — история карточки
- `idx_perf_session` ON `(session_id)` — результаты сессии

---

## Table: `agent_configs`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | |
| `provider` | TEXT | NOT NULL, UNIQUE | `anthropic` / `openai` |
| `status` | TEXT | NOT NULL | `connected` / `expired` / `error` / `not_configured` |
| `provider_account_id` | TEXT | nullable | ID аккаунта у провайдера (из токена) |
| `created_at` | DATETIME | NOT NULL, default now() | |
| `updated_at` | DATETIME | NOT NULL, default now() | |

**Примечание**: сами учётные данные (API-ключ / OAuth-токен) хранятся **только** в зашифрованном
файле `~/.config/study-kit/credentials.json.enc`, не в SQLite.

---

## Table: `processing_jobs`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | TEXT | PK | ARQ job ID |
| `source_file_id` | UUID | FK → source_files, NOT NULL | |
| `status` | TEXT | NOT NULL | `queued` / `processing` / `completed` / `failed` |
| `progress_pct` | INTEGER | NOT NULL, default 0 | 0–100 |
| `stage` | TEXT | nullable | `extracting_text` / `chunking` / `generating_items` |
| `created_at` | DATETIME | NOT NULL, default now() | |
| `updated_at` | DATETIME | NOT NULL, default now() | |

---

## State Transitions

### SourceFile.processing_status
```
queued → processing → completed
                    → failed
```

### KnowledgeItem.state (FSRS)
```
new → learning → review
                ← relearning (после Again на зрелой карточке)
learning → relearning (если провал на learning шаге после graduation)
```

### AgentConfig.status
```
not_configured → connected → expired → connected (после refresh)
                           → error
```

---

## Indexing Plan (по принципу Performance конституции)

| Запрос | Индекс | Причина |
|--------|--------|---------|
| Карточки урока на сегодня | `(lesson_id, due_date)` | Основной путь сессии |
| Глобальная очередь повторений | `(due_date, state)` | Dashboard "due today" |
| История повторений карточки | `(knowledge_item_id, reviewed_at DESC)` | Profile карточки |
| Результаты сессии | `(session_id)` | Экран результатов |
| Файлы урока | `(lesson_id, processing_status)` | Список файлов |
| Уроки в пространстве | `(workspace_id, order_index)` | Навигация |
