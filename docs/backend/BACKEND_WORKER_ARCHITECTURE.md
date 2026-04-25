# 🤖 Архитектура воркеров (Worker Architecture)

## Обзор

**Воркер** — внешний Python-скрипт, который выполняется на Windows-машине и обрабатывает задачи генерации озвучки.

**Принципы:**
- Нет блокировок в БД — нет `worker_id`, `locked_at`, `retry_count`
- Взаимодействие через REST API (поллинг)
- Один воркер обрабатывает один статус запуск
- Можно запустить несколько воркеров параллельно на разные статусы

---

## Архитектура

```
┌─────────────────────────────────────────────────────────┐
│                    Worker (Windows)                     │
│                                                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐ │
│  │   Polling   │ →  │  Processing │ →  │   Update    │ │
│  │   (10 sec)  │    │   (Logic)   │    │  (API call) │ │
│  └─────────────┘    └─────────────┘    └─────────────┘ │
└─────────────────────────────────────────────────────────┘
                            ↓
                    REST API (FastAPI)
                            ↓
                   ┌─────────────────┐
                   │    Backend      │
                   │  (SQLite + S3)  │
                   └─────────────────┘
```

---

## Параметры запуска

**Команда:**
```bash
python worker.py --status <status> [options]
```

**Обязательные параметры:**
- `--status` — статус задач для обработки: `initial` | `text_templated` | `accented` | `validated` | `splitted`

**Опциональные параметры:**
- `--replica-kind` — фильтр по типу реплики: `quest` | `gossip`
- `--locale` — фильтр по языку: `ruRU` | `enGB`
- `--limit` — количество задач за один цикл (0 = все, default: 50)
- `--narrator-voice-id` — ID голоса диктора (для воркера на этапе `splitted`)
- `--poll-interval` — интервал опроса API в секундах (default: 10)

---

## Примеры запуска

### 1. Воркер шаблонизации текста
```bash
python worker.py --status text_templated --replica-kind quest --locale ruRU --limit 50
```
**Логика:**
- Получает 50 задач со статусом `text_templated`
- Обрабатывает: заменяет переменные WoW, числа, спецсимволы
- Обновляет статус в `accented`

### 2. Воркер простановки ударений
```bash
python worker.py --status accented --locale ruRU --limit 100
```
**Логика:**
- Получает 100 задач со статусом `accented`
- Применяет `ruaccent` для простановки ударений
- Обновляет статус в `validated`

### 3. Воркер валидации текста
```bash
python worker.py --status validated --locale ruRU --limit 0
```
**Логика:**
- Получает все задачи со статусом `validated`
- Проверяет на отсутствие нерусских символов
- Обновляет статус в `splitted` (или `error`)

### 4. Воркер создания батчей (с диктором)
```bash
python worker.py --status splitted --narrator-voice-id=99 --limit 50
```
**Логика:**
- Получает 50 задач со статусом `splitted`
- Разбивает текст на фрагменты
- Создаёт `GENERATION_BATCHES`
- Для блоков `<...>` использует голос диктора ID=99
- Обновляет статус в `merged` (когда все батчи созданы)

### 5. Воркер склейки аудио
```bash
python worker.py --status merged --limit 20
```
**Логика:**
- Получает 20 готовых генераций через `GET /generations/ready-for-merge`
- Скачивает все батчи из S3
- Склеивает по `order_index`
- Загружает финальный OGG в S3
- Обновляет статус в `merged`

---

## Логика воркера (псевдокод)

```python
def worker(status, replica_kind, locale, limit, poll_interval=10):
    while True:
        # 1. Получение задач
        tasks = api.get_tasks(
            status=status,
            replica_kind=replica_kind,
            locale=locale,
            limit=limit
        )
        
        if not tasks:
            time.sleep(poll_interval)
            continue
        
        # 2. Обработка
        for task in tasks:
            try:
                result = process_task(task, status)
                
                # 3. Обновление статуса
                api.update_status(
                    id=task.id,
                    status=result.next_status,
                    text=result.text,  # если изменился
                    error=result.error  # если ошибка
                )
                
            except Exception as e:
                api.update_status(
                    id=task.id,
                    status='error',
                    error=str(e)
                )
```

---

## Обработка ошибок

**При возникновении ошибки:**
1. Перевод задачи в статус `error`
2. Запись текста ошибки в поле `error`
3. Логирование в локальный файл воркера
4. Запись в `GENERATIONS.history` (автоматически через API)

**Возврат из ошибки:**
- Автоматического возврата нет
- Требует ручного вмешательства через API:
  ```bash
  PUT /generations/{id}/status
  {"status": "text_templated"}
  ```

---

## Масштабирование

### Стратегия 1: Разделение по статусам

Запустить отдельные воркеры на каждый статус:
```bash
# Воркер 1: text_templated → accented
python worker.py --status text_templated --limit 100

# Воркер 2: accented → validated
python worker.py --status accented --limit 100

# Воркер 3: validated → splitted
python worker.py --status validated --limit 100
```

**Преимущества:**
- Параллельная обработка разных этапов
- Легко масштабировать узкие места

### Стратегия 2: Разделение по типам реплик

```bash
# Воркер 1: квесты
python worker.py --status text_templated --replica-kind quest --limit 50

# Воркер 2: gossip
python worker.py --status text_templated --replica-kind gossip --limit 50
```

### Стратегия 3: Разделение по языкам

```bash
# Воркер 1: русский язык
python worker.py --status accented --locale ruRU --limit 100

# Воркер 2: английский язык
python worker.py --status accented --locale enGB --limit 100
```

---

## Требования к окружению

**Платформа:** Windows (для GPU-доступа)

**Зависимости:**
- Python 3.9+
- Библиотеки для TTS (Silero, Coqui, и т.д.)
- Библиотеки для работы с аудио (pydub, ffmpeg)
- HTTP-клиент (requests, httpx)

**Конфигурация:**
```ini
# .env
BACKEND_URL=https://api.voiceovercraft.local
S3_ENDPOINT=https://s3.voiceovercraft.local
S3_ACCESS_KEY=xxx
S3_SECRET_KEY=xxx
NARRATOR_VOICE_ID=99
POLL_INTERVAL=10
```

---

## Мониторинг

**Метрики для отслеживания:**
- Количество обработанных задач за цикл
- Среднее время обработки одной задачи
- Процент ошибок
- Время простоя (когда задач нет)

**Логирование:**
- Каждая задача: ID, статус до/после, время обработки
- Ошибки: ID задачи, текст ошибки, стек трейс
- Статистика за сессию

---

## 🔗 Связанные документы

- [`BACKEND_GENERATION_STAGES.md`](./BACKEND_GENERATION_STAGES.md) — Детальное описание этапов
- [`BACKEND_API_REFERENCE.md`](./BACKEND_API_REFERENCE.md) — API endpoints
- [`BACKEND_VOICE_LOGIC.md`](./BACKEND_VOICE_LOGIC.md) — Выбор голосов (параметр `--narrator-voice-id`)
