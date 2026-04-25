# 🎤 Логика выбора голосов (Voice Selection)

## Приоритет выбора (сверху вниз)

1. **Голос для GENERATION** — `GENERATIONS.specific_voice_id`
   - Явно указан для конкретной генерации
   - Наивысший приоритет

2. **Голос для REPLICS** — `REPLICS.specific_voice_id`
   - Явно указан для реплики
   - Применяется ко всем генерациям этой реплики

3. **Голос для SPEAKERS** — `SPEAKERS.specific_voice_id`
   - Явно указан для говорителя
   - Применяется ко всем репликам этого говорителя

4. **Фоллбек** — стандартный голос `{race}_{gender}_{NPCSoundID}`
   - Автоматическая генерация имени голоса
   - Пример: `human_male_123`

---

## Голос диктора (Narrator Voice)

**Описание:** Специальный голос для озвучки блоков текста в символах `<...>`.

**Настройка:**
- Передаётся в **конфиге воркера**: `--narrator-voice-id=42`
- Применяется ко всем батчам, содержащим текст в формате `<...>`

**Пример:**
```
Текст: "<Это говорит диктор.> Герой, ты должен убить пять волков!"
Результат:
  - Блок "<Это говорит диктор.>" → голос диктора (ID=42)
  - Остальной текст → по приоритету выше
```

---

## Алгоритм выбора голоса для батча

**На этапе:** `splitted` (создание `GENERATION_BATCHES`)

**Шаги:**

1. **Проверка на блок диктора:**
   - Если текст батча начинается с `<` и заканчивается на `>` → использовать `narrator_voice_id`
   - Иначе → продолжить по приоритету

2. **Приоритет 1:** `GENERATIONS.specific_voice_id`
   - Если не `NULL` → использовать этот голос

3. **Приоритет 2:** `REPLICS.specific_voice_id`
   - Получить `REPLICS` по `GENERATIONS.replicaID`
   - Если `specific_voice_id` не `NULL` → использовать этот голос

4. **Приоритет 3:** `SPEAKERS.specific_voice_id`
   - Получить `SPEAKERS` по `REPLICS.speakerID`
   - Если `specific_voice_id` не `NULL` → использовать этот голос

5. **Фоллбек:** `{race}_{gender}_{NPCSoundID}`
   - Получить `race`, `gender`, `NPCSoundID` из `SPEAKERS`
   - Сформировать имя голоса: `{race}_{gender}_{NPCSoundID}`
   - Найти `VOICES` по имени
   - Если не найден → ошибка (нет подходящего голоса)

---

## Примеры

### Пример 1: Индивидуальный голос для генерации

```
GENERATIONS.specific_voice_id = 42
Результат: голос ID=42 (независимо от остальных настроек)
```

### Пример 2: Голос для реплики

```
GENERATIONS.specific_voice_id = NULL
REPLICS.specific_voice_id = 15
Результат: голос ID=15
```

### Пример 3: Голос для говорителя

```
GENERATIONS.specific_voice_id = NULL
REPLICS.specific_voice_id = NULL
SPEAKERS.specific_voice_id = 7
Результат: голос ID=7
```

### Пример 4: Фоллбек

```
GENERATIONS.specific_voice_id = NULL
REPLICS.specific_voice_id = NULL
SPEAKERS.specific_voice_id = NULL
SPEAKERS: race=human, gender=male, NPCSoundID=123
Результат: голос с именем "human_male_123"
```

### Пример 5: Блок диктора

```
Батч: "<Вступительное слово.> Далее основной текст."
narrator_voice_id = 99
Результат:
  - "<Вступительное слово.>" → голос ID=99
  - "Далее основной текст." → по приоритету (1-4)
```

---

## Таблица VOICES

**Модель:** [`BACKEND_DATA_MODELS.md`](./BACKEND_DATA_MODELS.md#4-voices-голоса-и-настройки)

| Поле | Тип | Описание |
|:---|:---|:---|
| `id` | PK | Уникальный идентификатор |
| `reference_file` | String | S3 URL файла-референса для AI-клонирования |
| `locale` | Enum | Язык: `ruRU` | `enGB` | ... |
| `name` | String | Уникальное имя (шаблон `{race}_{gender}_{NPCSoundID}` или кастомное) |
| `created_at`, `updated_at` | Timestamp | Время создания/обновления |

---

## Рекомендации по настройке

### 1. Предзаполнение VOICES

Создать голоса для всех комбинаций `{race}_{gender}_{NPCSoundID}`:
- Расы: `human`, `dwarf`, `gnome`, `night_elf`, `orc`, `undead`, `tauren`, `troll`
- Гендеры: `male`, `female`
- NPCSoundID: из базы cmangos (например, 1-500)

### 2. Кастомные голоса

Для уникальных NPC использовать индивидуальное имя:
```
name: "thrall_unique"
reference_file: s3://bucket/voices/thrall_reference.wav
```

### 3. Голос диктора

Создать отдельный голос с стабильным качеством:
```
id: 99
name: "narrator_ru"
locale: ruRU
reference_file: s3://bucket/voices/narrator_reference.wav
```

---

## 🔗 Связанные документы

- [`BACKEND_DATA_MODELS.md`](./BACKEND_DATA_MODELS.md) — Модель данных (таблица VOICES)
- [`BACKEND_GENERATION_STAGES.md`](./BACKEND_GENERATION_STAGES.md) — Этап `splitted` (создание батчей)
- [`BACKEND_WORKER_ARCHITECTURE.md`](./BACKEND_WORKER_ARCHITECTURE.md) — Параметры воркера (`--narrator-voice-id`)
