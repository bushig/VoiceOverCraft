# 🔄 Очередь задач (Queue System)

## Архитектура

**Принципы:**
- Минимум компонентов — только PostgreSQL + FastAPI
- Нет блокировок в БД — нет `worker_id`, `locked_at`, `retry_count`
- Воркер — внешний Python-скрипт на Windows
- Взаимодействие через REST API
- Файлы: Воркер → Бэкенд → S3

```
┌─────────────┐      REST API      ┌──────────────┐
│   Backend   │ ←────────────────→ │    Worker    │
│  (FastAPI)  │                    │  (Python)    │
│             │                    │              │
│  - Фильтры  │                    │  - Поллинг   │
│  - Статусы  │                    │  - Логика    │
│  - S3       │                    │  - Загрузка  │
└──────┬──────┘                    └──────────────┘
       │
       ↓
┌─────────────┐
│ PostgreSQL  │
└─────────────┘
```

---

## API Endpoints

### 1. Получение задач

`GET /api/v1/generations/tasks`

**Параметры:**
- `status` (required) — `text_prepare` | `accent` | `generation` | `merging`
- `replica_kind` (optional) — `quest` | `gossip`
- `locale` (optional) — `ruRU` | `enGB`
- `limit` (optional) — количество задач (0 = все по фильтрам)

**Ответ:** Список задач с полями: `id`, `replicaID`, `status`, `text`, `locale`, `specific_voice_id`, `target_gender`

---

### 2. Обновление статуса задачи

`PUT /api/v1/generations/{id}/status`

**Тело:**
- `status` (required) — новый статус
- `text` (optional) — обновлённый текст (если изменился)
- `error` (optional) — текст ошибки (если `status=error`)

---

### 3. Загрузка файла

`POST /api/v1/generations/{id}/file` (multipart/form-data)

**Тело:** `file` — аудиофайл в формате OGG

**Процесс:** Воркер → Бэкенд → S3 → Обновление `generated_file` в БД

**Ответ:** `{ "success": true, "file_url": "S3_URL" }`

---

## Диаграмма статусов

```
initial → text_prepare → accent → generation → merging → done
                                              ↓
                                            error
```

**Правила переходов:**
- Любой статус → следующий по цепочке
- Любой статус → `error` (при ошибке)
- Из `error` → нет автоматического возврата (только вручную через API)
- `merging` → `done` (финал успешной генерации)

**Описание этапов:**

| Статус | Описание |
|:---|:---|
| `initial` | Задача создана, ожидает обработки |
| `text_prepare` | Подготовка текста (очистка, шаблонизация) |
| `accent` | Простановка ударений в тексте |
| `generation` | AI-генерация аудио по тексту |
| `merging` | Сборка финального аудио из фрагментов |
| `done` | Генерация завершена успешно |
| `error` | Ошибка на одном из этапов |

---

## История (History)

**Поле:** `GENERATIONS.history` (тип `jsonb`)

**Формат записи:**
```json
{
  "timestamp": "2026-04-25T10:00:00Z",
  "from_status": "initial",
  "to_status": "text_prepare",
  "event_type": "transition"
}
```

**Для ошибок:**
```json
{
  "timestamp": "2026-04-25T10:05:00Z",
  "from_status": "text_prepare",
  "to_status": "error",
  "event_type": "error",
  "error_message": "Текст ошибки"
}
```

**Что записывается:**
- ✅ Все переходы статусов (`event_type: transition`)
- ✅ Все ошибки (`event_type: error`)

**Пример истории:**
```json
[
  {
    "timestamp": "2026-04-25T10:00:00Z",
    "from_status": "initial",
    "to_status": "text_prepare",
    "event_type": "transition"
  },
  {
    "timestamp": "2026-04-25T10:05:00Z",
    "from_status": "text_prepare",
    "to_status": "accent",
    "event_type": "transition"
  },
  {
    "timestamp": "2026-04-25T10:10:00Z",
    "from_status": "accent",
    "to_status": "error",
    "event_type": "error",
    "error_message": "Сервис ударений недоступен"
  }
]
```

---

## Запуск воркера

Воркер — внешний Python-скрипт, который запускается с параметрами:

```bash
# Обработать все квесты в статусе text_prepare
python worker.py --status text_prepare --replica-kind quest --locale ruRU --limit 0

# Обработать 50 задач gossip в статусе accent
python worker.py --status accent --replica-kind gossip --locale enGB --limit 50

# Обработать все задачи генерации для ruRU
python worker.py --status generation --locale ruRU --limit 0
```

**Параметры:**
- `--status` (required) — один статус для обработки
- `--replica-kind` (optional) — фильтр по типу реплики
- `--locale` (optional) — фильтр по языку
- `--limit` (optional, default: 50) — количество задач (0 = все)

**Логика воркера:**
1. Поллинг API с интервалом ~10 сек
2. Получение задач по фильтрам (`status`, `replica_kind`, `locale`, `limit`)
3. Обработка согласно статусу (очистка текста, ударения, генерация аудио, склейка)
4. Обновление статуса через API
5. Загрузка файла (для `merging`) через `POST /generations/{id}/file`
6. Обработка ошибок: перевод в `status=error`

---

## Технические детали

### Формат файлов
- **Только OGG** — все аудиофайлы конвертируются в OGG перед загрузкой
- **S3 Storage** — бэкенд хранит файлы в S3-совместимом хранилище

### Сортировка задач
- `ORDER BY created_at ASC` — FIFO (первым создан, первым обработан)

### Лимит задач
- `limit=0` — вернуть все задачи по фильтрам
- `limit=N` — вернуть максимум N задач

### Масштабирование
- Можно запустить несколько воркеров с разными параметрами
- Пример: один воркер обрабатывает `text_prepare`, другой — `generation`
