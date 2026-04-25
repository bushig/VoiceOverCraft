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

```http
GET /api/v1/generations/tasks
```

**Параметры:**
- `status` (required) — один статус: `text_prepare` | `accent` | `generation` | `merging`
- `replica_kind` (optional) — `quest` | `gossip`
- `locale` (optional) — `ruRU` | `enGB`
- `limit` (optional) — количество задач (0 = все по фильтрам)

**Ответ:**
```json
[
  {
    "id": 123,
    "replicaID": 456,
    "status": "text_prepare",
    "text": "Текст реплики",
    "locale": "ruRU",
    "specific_voice_id": null,
    "target_gender": "male"
  }
]
```

---

### 2. Обновление статуса задачи

```http
PUT /api/v1/generations/{id}/status
Content-Type: application/json
```

**Тело запроса:**
```json
{
  "status": "accent",
  "text": "обработанный текст",
  "error": "текст ошибки"
}
```

**Поля:**
- `status` (required) — новый статус
- `text` (optional) — обновлённый текст (если изменился)
- `error` (optional) — текст ошибки (если `status=error`)

**Примеры:**

Успех:
```json
{
  "status": "accent",
  "text": "Подготовленный текст с ударениями"
}
```

Ошибка:
```json
{
  "status": "error",
  "error": "Не удалось проставить ударения: текст пуст"
}
```

---

### 3. Загрузка файла

```http
POST /api/v1/generations/{id}/file
Content-Type: multipart/form-data
```

**Тело:**
- `file` (required) — аудиофайл в формате OGG

**Процесс:**
1. Воркер отправляет файл бэкэнду
2. Бэкенд загружает файл в S3
3. Бэкенд обновляет поле `generated_file` в БД
4. Бэкенд возвращает подтверждение

**Ответ:**
```json
{
  "success": true,
  "file_url": "https://s3.example.com/generations/123/audio.ogg"
}
```

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

## Пример воркера (Python)

```python
import requests
import time
from pathlib import Path

API_URL = "http://localhost:8000/api/v1"

# Параметры запуска (настраиваются владельцем)
STATUS = "text_prepare"
REPLICA_KIND = "quest"
LOCALE = "ruRU"
LIMIT = 50
POLL_INTERVAL = 10  # секунд


def process_task(task):
    """
    Выполнить логику для текущего статуса задачи.
    Возвращает результат и путь к файлу (если нужен).
    """
    status = task['status']
    
    if status == 'text_prepare':
        # Подготовка текста: очистка, шаблонизация
        processed_text = clean_text(task['text'])
        return {'text': processed_text}
    
    elif status == 'accent':
        # Простановка ударений
        accented_text = add_accents(task['text'])
        return {'text': accented_text}
    
    elif status == 'generation':
        # AI-генерация аудио
        audio_path = generate_audio(task['text'], task['locale'], task['specific_voice_id'])
        return {'file_path': audio_path}
    
    elif status == 'merging':
        # Сборка финального аудио из фрагментов
        final_audio = merge_audio(task['generation_id'])
        return {'file_path': final_audio}
    
    else:
        raise ValueError(f"Неизвестный статус: {status}")


def get_next_status(current_status):
    """Вернуть следующий статус по цепочке."""
    chain = {
        'initial': 'text_prepare',
        'text_prepare': 'accent',
        'accent': 'generation',
        'generation': 'merging',
        'merging': 'done'
    }
    return chain[current_status]


def main():
    while True:
        try:
            # 1. Получить задачи
            resp = requests.get(f"{API_URL}/generations/tasks", params={
                "status": STATUS,
                "replica_kind": REPLICA_KIND,
                "locale": LOCALE,
                "limit": LIMIT
            })
            resp.raise_for_status()
            tasks = resp.json()
            
            if not tasks:
                print(f"Задач со статусом '{STATUS}' не найдено")
                time.sleep(POLL_INTERVAL)
                continue
            
            print(f"Получено задач: {len(tasks)}")
            
            # 2. Обработать каждую задачу
            for task in tasks:
                try:
                    result = process_task(task)
                    
                    # 3. Обновить статус
                    next_status = get_next_status(task['status'])
                    update_payload = {
                        "status": next_status,
                    }
                    
                    # Добавить текст, если изменился
                    if 'text' in result:
                        update_payload['text'] = result['text']
                    
                    resp = requests.put(
                        f"{API_URL}/generations/{task['id']}/status",
                        json=update_payload
                    )
                    resp.raise_for_status()
                    
                    # 4. Если это был merging — загрузить файл
                    if task['status'] == 'merging' and 'file_path' in result:
                        with open(result['file_path'], 'rb') as f:
                            resp = requests.post(
                                f"{API_URL}/generations/{task['id']}/file",
                                files={'file': f}
                            )
                            resp.raise_for_status()
                            print(f"Файл загружен для задачи {task['id']}")
                    
                    print(f"Задача {task['id']} переведена в статус '{next_status}'")
                    
                except Exception as e:
                    # 5. При ошибке
                    print(f"Ошибка обработки задачи {task['id']}: {e}")
                    requests.put(
                        f"{API_URL}/generations/{task['id']}/status",
                        json={
                            "status": "error",
                            "error": str(e)
                        }
                    )
            
            time.sleep(POLL_INTERVAL)
            
        except requests.RequestException as e:
            print(f"Ошибка соединения с API: {e}")
            time.sleep(POLL_INTERVAL)
        except KeyboardInterrupt:
            print("Остановка воркера")
            break


if __name__ == "__main__":
    main()
```

---

## Запуск воркера

Владелец системы запускает воркер с нужными параметрами:

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
