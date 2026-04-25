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
1. Замена переменных WoW на универсальные значения:
   ```python
   REPLACE_DICT = {
       '$b': '\n',
       '$B': '\n',
       '$n': 'герой',
       '$N': 'Герой',
       '$C': 'Путник',
       '$c': 'путник',
       '$R': 'Странник',
       '$r': 'странник'
   }
   ```

2. Замена гендерных маркеров (TODO: уточнить синтаксис WoW)

3. Замена цифр на слова:
   - `5` → `"пять"`
   - `10` → `"десять"`

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
   ```python
   batches = []
   for i, fragment in enumerate(fragments):
       voice_id = select_voice(fragment, generation)
       batches.append({
           'generation_id': generation.id,
           'voice_id': voice_id,
           'text': fragment,
           'order_index': i,
           'status': 'pending'
       })
   ```

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

1. **Получение готовых генераций:**
   ```http
   GET /api/v1/generations/ready-for-merge
   ```
   
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

```http
GET /api/v1/generations/ready-for-merge
```

**Ответ:**
```json
[
  {
    "id": 123,
    "locale": "ruRU",
    "batches": [
      {
        "id": 1,
        "order_index": 0,
        "generated_file": "https://s3.example.com/batches/123_0.ogg",
        "text": "Прив+ет гер+ой!"
      },
      {
        "id": 2,
        "order_index": 1,
        "generated_file": "https://s3.example.com/batches/123_1.ogg",
        "text": "Я р+ад тебя в+идеть."
      }
    ]
  }
]
```

---

### 2. Обновление статуса GENERATION

```http
PUT /api/v1/generations/{id}/status
Content-Type: application/json
```

**Тело:**
```json
{
  "status": "merged",
  "generated_file": "https://s3.example.com/generations/123/final.ogg"
}
```

---

### 3. Обновление статуса BATCH

```http
PUT /api/v1/generation-batches/{id}/status
Content-Type: application/json
```

**Тело:**
```json
{
  "status": "complete",
  "generated_file": "https://s3.example.com/batches/456.ogg"
}
```

---

### 4. Загрузка файла в S3

```http
POST /api/v1/files/upload
Content-Type: multipart/form-data
```

**Тело:**
- `file` — аудиофайл в формате OGG

**Ответ:**
```json
{
  "success": true,
  "file_url": "https://s3.example.com/uploads/abc123.ogg"
}
```

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

## 💻 Примеры воркеров

### Воркер для акцентов

```python
import requests

API_URL = "http://localhost:8000/api/v1"

def add_accents(text):
    """Проставить ударения через ruaccent."""
    from ruaccent import add_accent
    return add_accent(text)

def main():
    # Получить задачи с проставленными ударениями
    resp = requests.get(f"{API_URL}/generations/tasks", params={
        "status": "accented",
        "limit": 50
    })
    tasks = resp.json()
    
    for task in tasks:
        try:
            # Проверить текст
            if has_invalid_chars(task['text']):
                raise ValueError("Найдены нерусские символы")
            
            # Перевести в validated
            requests.put(f"{API_URL}/generations/{task['id']}/status", json={
                "status": "validated"
            })
            print(f"Задача {task['id']} проверена")
            
        except Exception as e:
            requests.put(f"{API_URL}/generations/{task['id']}/status", json={
                "status": "error",
                "error": str(e)
            })
```

---

### Воркер для синтеза батчей

```python
import requests

API_URL = "http://localhost:8000/api/v1"
NARRATOR_VOICE_ID = 42  # из конфига

def generate_audio(text, voice_id):
    """AI-генерация аудио через F5 TTS."""
    # Здесь логика вызова F5 TTS
    # Возвращает путь к локальному OGG файлу
    pass

def select_voice(text, generation):
    """Выбрать голос для батча."""
    if text.startswith('<') and text.endswith('>'):
        return NARRATOR_VOICE_ID
    
    # Приоритет голосов
    if generation.get('specific_voice_id'):
        return generation['specific_voice_id']
    # ... остальная логика
    return fallback_voice_id

def main():
    # Получить батчи для генерации
    resp = requests.get(f"{API_URL}/generation-batches/tasks", params={
        "status": "pending",
        "limit": 100
    })
    batches = resp.json()
    
    # Группировка по voice_id (оптимизация)
    from itertools import groupby
    batches_by_voice = groupby(batches, key=lambda b: b['voice_id'])
    
    for voice_id, voice_batches in batches_by_voice:
        for batch in voice_batches:
            try:
                # Сгенерировать аудио
                audio_path = generate_audio(batch['text'], voice_id)
                
                # Загрузить в S3 через бэкэнд
                with open(audio_path, 'rb') as f:
                    upload_resp = requests.post(
                        f"{API_URL}/files/upload",
                        files={'file': f}
                    )
                    s3_url = upload_resp.json()['file_url']
                
                # Обновить статус батча
                requests.put(f"{API_URL}/generation-batches/{batch['id']}/status", json={
                    "status": "complete",
                    "generated_file": s3_url
                })
                print(f"Батч {batch['id']} сгенерирован")
                
            except Exception as e:
                requests.put(f"{API_URL}/generation-batches/{batch['id']}/status", json={
                    "status": "error",
                    "error": str(e)
                })
```

---

### Воркер для склейки (merge)

```python
import requests
from pydub import AudioSegment
import tempfile
import os

API_URL = "http://localhost:8000/api/v1"

def merge_audio_files(file_paths, output_path):
    """Склеить аудиофайлы в порядке order_index."""
    combined = AudioSegment.empty()
    for file_path in sorted(file_paths):
        audio = AudioSegment.from_ogg(file_path)
        combined += audio
    combined.export(output_path, format='ogg')
    return output_path

def main():
    # Получить генерации, готовые к склейке
    resp = requests.get(f"{API_URL}/generations/ready-for-merge")
    generations = resp.json()
    
    for gen in generations:
        try:
            # Скачать все батчи
            batch_files = []
            for batch in sorted(gen['batches'], key=lambda b: b['order_index']):
                resp = requests.get(batch['generated_file'])
                temp_file = tempfile.NamedTemporaryFile(delete=False, suffix='.ogg')
                temp_file.write(resp.content)
                batch_files.append(temp_file.name)
            
            # Склеить
            output_file = tempfile.NamedTemporaryFile(delete=False, suffix='.ogg')
            merge_audio_files(batch_files, output_file.name)
            
            # Загрузить в S3
            with open(output_file.name, 'rb') as f:
                upload_resp = requests.post(
                    f"{API_URL}/files/upload",
                    files={'file': f}
                )
                s3_url = upload_resp.json()['file_url']
            
            # Перевести в merged
            requests.put(f"{API_URL}/generations/{gen['id']}/status", json={
                "status": "merged",
                "generated_file": s3_url
            })
            print(f"Генерация {gen['id']} завершена")
            
            # Очистка
            for f in batch_files:
                os.unlink(f)
            os.unlink(output_file.name)
            
        except Exception as e:
            requests.put(f"{API_URL}/generations/{gen['id']}/status", json={
                "status": "error",
                "error": str(e)
            })
```

---

## 📝 Скрипт создания initial задач

**Файл:** `scripts/create_initial_generations.py`

**Пример запуска:**
```bash
python scripts/create_initial_generations.py \
  --zone 1,2,3 \
  --min-level 10 \
  --max-level 80 \
  --replica-kind quest \
  --replica-type accept,complete \
  --extension vanilla
```

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
