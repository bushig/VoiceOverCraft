# 🔌 API Reference

## Базовый URL

`/api/v1`

---

## Endpoints для работы с задачами генерации

### 1. Получение готовых к склейке генераций

**Endpoint:** `GET /generations/ready-for-merge`

**Описание:** Возвращает генерации со статусом `splitted`, у которых все связанные батчи имеют `status='complete'`.

**Параметры запроса:** Нет

**Ответ:** `200 OK`
```json
[
  {
    "id": 123,
    "replicaID": 456,
    "status": "splitted",
    "locale": "ruRU",
    "target_gender": "male",
    "batches": [
      {
        "id": 789,
        "order_index": 0,
        "generated_file": "s3://bucket/audio/batch_789.ogg"
      },
      {
        "id": 790,
        "order_index": 1,
        "generated_file": "s3://bucket/audio/batch_790.ogg"
      }
    ]
  }
]
```

---

### 2. Получение задач для воркера

**Endpoint:** `GET /generations/tasks`

**Описание:** Получение списка задач для обработки воркером.

**Параметры запроса:**
- `status` (required) — статус задач: `initial` | `text_templated` | `accented` | `validated` | `splitted`
- `replica_kind` (optional) — тип реплики: `quest` | `gossip`
- `locale` (optional) — язык: `ruRU` | `enGB`
- `limit` (optional) — количество задач (0 = все по фильтрам, default: 50)

**Пример запроса:**
```
GET /generations/tasks?status=text_templated&replica_kind=quest&locale=ruRU&limit=50
```

**Ответ:** `200 OK`
```json
[
  {
    "id": 123,
    "replicaID": 456,
    "status": "text_templated",
    "text": "Герой, ты должен убить пять волков!",
    "locale": "ruRU",
    "target_gender": "male",
    "specific_voice_id": 42
  }
]
```

**Сортировка:** `ORDER BY created_at ASC` (FIFO)

---

### 3. Обновление статуса GENERATION

**Endpoint:** `PUT /generations/{id}/status`

**Описание:** Обновление статуса задачи генерации.

**Параметры пути:**
- `id` (required) — ID генерации

**Тело запроса:**
```json
{
  "status": "text_templated",
  "text": "обновлённый текст (optional)",
  "error": "текст ошибки (optional, если status=error)"
}
```

**Поля:**
- `status` (required) — новый статус: `initial` | `text_templated` | `accented` | `validated` | `splitted` | `merged` | `error`
- `text` (optional) — обновлённый текст (если изменился на текущем этапе)
- `error` (optional) — текст ошибки (если `status=error`)

**Ответ:** `200 OK`
```json
{
  "id": 123,
  "status": "text_templated",
  "updated_at": "2026-04-25T10:00:00Z"
}
```

---

### 4. Обновление статуса GENERATION_BATCH

**Endpoint:** `PUT /generation-batches/{id}/status`

**Описание:** Обновление статуса батча генерации.

**Параметры пути:**
- `id` (required) — ID батча

**Тело запроса:**
```json
{
  "status": "complete",
  "generated_file": "s3://bucket/audio/batch_789.ogg"
}
```

**Поля:**
- `status` (required) — новый статус: `pending` | `complete` | `error`
- `generated_file` (optional) — S3 URL аудиофайла (если `status=complete`)

**Ответ:** `200 OK`
```json
{
  "id": 789,
  "status": "complete",
  "generated_file": "s3://bucket/audio/batch_789.ogg",
  "updated_at": "2026-04-25T10:05:00Z"
}
```

---

### 5. Загрузка аудиофайла в S3

**Endpoint:** `POST /generations/{id}/file`

**Описание:** Загрузка аудиофайла для генерации. Бэкенд загружает файл в S3 и обновляет `generated_file` в БД.

**Параметры пути:**
- `id` (required) — ID генерации или батча

**Content-Type:** `multipart/form-data`

**Тело запроса:**
- `file` (required) — аудиофайл в формате OGG

**Ответ:** `200 OK`
```json
{
  "success": true,
  "file_url": "s3://bucket/audio/generation_123.ogg"
}
```

---

## Endpoints для работы с голосами

### 6. Получение списка голосов

**Endpoint:** `GET /voices`

**Описание:** Получение списка доступных голосов.

**Параметры запроса:**
- `locale` (optional) — фильтр по языку: `ruRU` | `enGB`
- `race` (optional) — фильтр по расе
- `gender` (optional) — фильтр по полу: `male` | `female`

**Ответ:** `200 OK`
```json
[
  {
    "id": 42,
    "name": "human_male_123",
    "locale": "ruRU",
    "reference_file": "s3://bucket/voices/human_male_123.wav"
  }
]
```

---

### 7. Получение голоса по ID

**Endpoint:** `GET /voices/{id}`

**Описание:** Получение деталей конкретного голоса.

**Параметры пути:**
- `id` (required) — ID голоса

**Ответ:** `200 OK`
```json
{
  "id": 42,
  "name": "human_male_123",
  "locale": "ruRU",
  "reference_file": "s3://bucket/voices/human_male_123.wav",
  "created_at": "2026-04-01T00:00:00Z"
}
```

---

## Endpoints для работы с репликами

### 8. Получение реплики по ID

**Endpoint:** `GET /replicas/{id}`

**Описание:** Получение деталей реплики.

**Параметры пути:**
- `id` (required) — ID реплики

**Ответ:** `200 OK`
```json
{
  "id": 456,
  "questID": 789,
  "speakerID": 101,
  "raw_texts": {
    "ruRU": "$N, ты должен убить 5 волков!",
    "enGB": "$N, you must kill 5 wolves!"
  },
  "replica_kind": "quest",
  "replica_type": "accept",
  "extension": "vanilla"
}
```

---

## Коды ошибок

**400 Bad Request:**
```json
{
  "error": "Invalid status transition",
  "details": "Cannot transition from 'initial' to 'merged'"
}
```

**404 Not Found:**
```json
{
  "error": "Generation not found",
  "id": 999
}
```

**500 Internal Server Error:**
```json
{
  "error": "S3 upload failed",
  "details": "Connection timeout"
}
```

---

## Переходы статусов

Допустимые переходы для `GENERATIONS.status`:

```
initial → text_templated → accented → validated → splitted → merged
    ↓            ↓             ↓            ↓           ↓         ↓
  error        error         error        error       error    error
```

**Правила:**
- Любой статус может перейти в `error`
- Из `error` нет автоматического возврата (только вручную через API)
- `merged` — финальный статус, переходов из него нет

---

## 🔗 Связанные документы

- [`BACKEND_GENERATION_STAGES.md`](./BACKEND_GENERATION_STAGES.md) — Детальное описание этапов генерации
- [`BACKEND_DATA_MODELS.md`](./BACKEND_DATA_MODELS.md) — Модель данных
- [`BACKEND_WORKER_ARCHITECTURE.md`](./BACKEND_WORKER_ARCHITECTURE.md) — Архитектура воркеров
