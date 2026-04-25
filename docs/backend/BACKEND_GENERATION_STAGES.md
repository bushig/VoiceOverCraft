# 🔄 Этапы генерации (Generation Stages)

## Обзор

Документ описывает поэтапный процесс генерации озвучки от создания задачи до готового аудиофайла.

### Цепочка статусов

```
initial → text_templated → accented → validated → splitted → merged
```

**Принципы:**

- Статус отражает **текущее состояние** задачи (что уже сделано)
- Воркер берёт задачи определённого статуса и выполняет действия для перехода в следующий
- `merged` — финальный статус (файл готов и загружен в S3)
- В любой момент возможен переход в `error`

---

## 📊 Детали этапов

### 1. `initial` — Задача создана

**Кто создает:** Отдельный скрипт `scripts/create_initial_generations.py`

**Входные данные:**
- `REPLICS.raw_texts` — сырые тексты реплик
- `SPEAKERS` — данные о говорителе (раса, пол, NPCSoundID)
- `QUESTS` — данные о квесте (зона, уровни)

**Логика:**
1. Скрипт выбирает реплики по фильтрам:
   - `zone` — зона квеста
   - `min_level`, `max_level` — диапазон уровней
   - `replica_kind` — `quest` / `gossip`
   - `replica_type` — `accept` / `complete` / `progress`
   - `extension` — дополнение WoW

2. Для каждой реплики определяется `isGenderSpecific`:
   - Если текст требует разных гендеров → создаются **две** задачи (male/female)
   - Иначе → **одна** задача (по умолчанию male)

3. Создаётся запись в `GENERATIONS`:
   - `status` = `initial`
   - `replicaID` — ссылка на реплику
   - `target_gender` — пол
   - `locale` — язык
   - `specific_voice_id` — опционально

**Результат:** Запись в БД, готовая к шаблонизации

---

### 2. `text_templated` — Текст шаблонизирован

**Воркер:** `python worker.py --status text_templated`

**Вход:** `GENERATIONS` со статусом `text_templated` (текст уже шаблонизирован)

**Логика:**
1. Замена переменных WoW на универсальные значения: `$N` → `"Герой"`, `$n` → `"герой"`, `$C/$c` → `"путник"`, `$R/$r` → `"странник"`
2. Замена гендерных маркеров (TODO: уточнить синтаксис WoW)
3. Замена цифр на слова: `5` → `"пять"`, `10` → `"десять"`
4. Удаление спецсимволов, непонятных для TTS
5. Замена имени игрока на универсальное обращение

**Результат:**
- Один очищенный текст в `GENERATIONS.text`
- Переход в статус: `accented`

**Пример:**
```
До: "$N, ты должен убить 5 волков!"
После: "Герой, ты должен убить пять волков!"
```

---

### 3. `accented` — Ударения проставлены

**Воркер:** `python worker.py --status accented`

**Вход:** `GENERATIONS` со статусом `accented` (ударения уже проставлены)

**Логика:**
- Для `locale=ruRU`: библиотека `ruaccent`
  - Формат: `прив+ет` (ударение через `+`)
- Для других языков: текст не меняется (пустышка)

**Результат:**
- Текст с ударениями в `GENERATIONS.text`
- Переход в статус: `validated`

**Пример:**
```
До: "Привет герой"
После: "Прив+ет гер+ой"
```

---

### 4. `validated` — Текст проверен

**Воркер:** `python worker.py --status validated`

**Вход:** `GENERATIONS` со статусом `validated` (текст уже проверен)

**Логика (v1):**
- Для `locale=ruRU`: проверка на отсутствие нерусских символов
  - Ошибка если найдены `[a-zA-Z]` в русском тексте
- Для других языков: проверка пропускается

**Логика (future):**
- Подключение LLM для вычитки текста
- Проверка на осмысленность, контекст

**Результат:**
- Если валидно → переход в `splitted`
- Если ошибка → переход в `error` с текстом ошибки

---

### 5. `splitted` — Разбито на батчи

**Воркер:** `python worker.py --status splitted`

**Вход:** `GENERATIONS` со статусом `splitted` (батчи уже созданы)

**Логика:**

1. **Разбиение текста на фрагменты:**
    - По предложениям (`.`, `!`, `?`)
    - Или по максимальной длине (например, 200 символов)

2. **Создание записей в `GENERATION_BATCHES`:**
    - Для каждого фрагмента создаётся запись с `generation_id`, `voice_id`, `text`, `order_index`, `status='pending'`

3. **Выбор голоса для каждого батча:**
    - Блок `<...>` (диктор) → `narrator_voice_id` (из конфига воркера)
    - Остальное → по приоритету (см. `BACKEND_VOICE_LOGIC.md`):
      1. Голос из `GENERATIONS.specific_voice_id`
      2. Голос из `REPLICS.specific_voice_id`
      3. Голос из `SPEAKERS.specific_voice_id`
      4. Фоллбек: `{race}_{gender}_{NPCSoundID}`

**Результат:**
- Массив `GENERATION_BATCHES` для данной `GENERATION`
- Переход в статус: `merged` (когда все батчи сгенерированы)

---

### 6. `merged` — Готово (склеено + загружено в S3)

**Воркер:** `python worker.py --status merged`

**Вход:** `GENERATIONS` готовые к склейке

**Логика:**

1. **Получение готовых генераций:** `GET /api/v1/generations/ready-for-merge`
    
    Бэкэнд возвращает `GENERATIONS` где:
    - `status = 'splitted'`
    - **Все** связанные батчи имеют `status = 'complete'`

2. **Склейка аудио:**
    - Скачать все батчи из S3 (по `generated_file`)
    - Склеить в порядке `order_index`
    - Нормализация громкости (опционально)

3. **Загрузка в S3:**
    - Загрузить финальный OGG-файл через API бэкэнда
    - Бэкэнд обновляет `GENERATIONS.generated_file` = S3 URL

**Результат:**
- Финальный аудиофайл в S3
- Статус: `merged` (финал)

---

## 🗄 Модель данных: GENERATION_BATCHES

| Поле | Тип | Описание |
|:---|:---|:---|
| `id` | PK | Уникальный идентификатор |
| `generation_id` | FK | Ссылка на `GENERATIONS.id` |
| `voice_id` | FK | Ссылка на `VOICES.id` (голос для этого батча) |
| `text` | Text | Фрагмент текста для озвучки |
| `order_index` | Integer | **Порядок склейки** (0, 1, 2, ...) |
| `status` | Enum | `pending` → `complete` → (error) |
| `generated_file` | String | **S3 URL** аудиофайла (после генерации) |
| `created_at` | Timestamp | Время создания |
| `updated_at` | Timestamp | Время обновления |

---

## 🔌 API Endpoints

### 1. Получение готовых к склейке генераций

`GET /api/v1/generations/ready-for-merge`

**Ответ:** Список генераций со статусом `splitted`, у которых все батчи имеют `status='complete'`. Каждая генерация содержит массив `batches` с `order_index` и `generated_file`.

---

### 2. Обновление статуса GENERATION

`PUT /api/v1/generations/{id}/status`

**Тело:** `{ "status": "merged", "generated_file": "S3_URL" }`

---

### 3. Обновление статуса BATCH

`PUT /api/v1/generation-batches/{id}/status`

**Тело:** `{ "status": "complete", "generated_file": "S3_URL" }`

---

### 4. Загрузка файла в S3

`POST /api/v1/files/upload` (multipart/form-data)

**Тело:** `file` — аудиофайл в формате OGG

**Ответ:** `{ "success": true, "file_url": "S3_URL" }`

---

## 🎯 Выбор голосов (Voice Selection)

### Приоритет (см. `BACKEND_VOICE_LOGIC.md`)

1. **Голос для GENERATION** — `GENERATIONS.specific_voice_id`
2. **Голос для REPLICS** — `REPLICS.specific_voice_id`
3. **Голос для SPEAKERS** — `SPEAKERS.specific_voice_id`
4. **Фоллбек** — стандартный голос `{race}_{gender}_{NPCSoundID}`

### Голос диктора

- Передаётся в **конфиге воркера**: `--narrator-voice-id=42`
- Применяется к блокам текста в символах `<...>`
- Пример: `"<Это говорит диктор.>"` → голос диктора

---

## 💻 Логика воркеров (кратко)

**Воркер акцентов:** Получает задачи со статусом `accented`, проверяет текст на отсутствие нерусских символов, переводит в `validated`. При ошибке — переход в `error`.

**Воркер синтеза батчей:** 
- Получает батчи со статусом `pending`
- Группирует по `voice_id` (оптимизация загрузки моделей)
- Для каждого батча: генерирует аудио → загружает в S3 → обновляет статус в `complete`
- Блоки `<...>` озвучиваются голосом диктора

**Воркер склейки (merge):**
- Получает генерации, где все батчи имеют `status='complete'`
- Скачивает батчи из S3, склеивает по `order_index`
- Загружает финальный OGG в S3, обновляет генерацию в `merged`

---

## 📝 Скрипт создания initial задач

**Файл:** `scripts/create_initial_generations.py`

**Параметры:** `--zone`, `--min-level`, `--max-level`, `--replica-kind`, `--replica-type`, `--extension`

**Логика:**
1. Выбрать `REPLICS` по фильтрам
2. Для каждой реплики:
    - Проверить `isGenderSpecific`
    - Создать одну или две `GENERATIONS` (male/female)
    - Установить `status = 'initial'`
3. Вывести статистику созданных задач

---

## 📊 Сводная таблица статусов

| Статус | Описание | Следующий этап | Воркер |
|:---|:---|:---|:---|
| `initial` | Задача создана | `text_templated` | Скрипт создания |
| `text_templated` | Текст шаблонизирован | `accented` | `--status text_templated` |
| `accented` | Ударения проставлены | `validated` | `--status accented` |
| `validated` | Текст проверен | `splitted` | `--status validated` |
| `splitted` | Батчи созданы | `merged` | `--status splitted` + синтез батчей |
| `merged` | **Готово** (файл в S3) | — | `--status merged` |

---

## 🔗 Связанные документы

- [`BACKEND_VOICE_LOGIC.md`](./BACKEND_VOICE_LOGIC.md) — Логика наследования голосов
- [`BACKEND_QUEUE_SYSTEM.md`](./BACKEND_QUEUE_SYSTEM.md) — Очередь задач и API
- [`BACKEND_DATA_MODELS.md`](./BACKEND_DATA_MODELS.md) — Модель данных
