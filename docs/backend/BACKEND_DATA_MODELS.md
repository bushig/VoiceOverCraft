# 🗄 Модель данных

## 1. `REPLICS` (Сырые реплики)

| Поле | Описание |
|:---|:---|
| `id` | PK |
| `questID` | ID квеста (ссылка на `QUESTS`), опционально |
| `speakerID` | ID говорителя (ссылка на `SPEAKERS`) |
| `raw_texts` | `jsonb` `{ "ruRU": "...", "enGB": "..." }` |
| `replica_kind` | `quest` / `gossip` |
| `replica_type` | `accept` / `complete` / `progress` |
| `textHashes` | `json` MD5-хеши по локалям |
| `extension` | Дополнение: `vanilla`, `burning_crusade`, и т.д. |
| `specific_voice_id` | Ссылка на `VOICES`, опционально |
| `created_at`, `updated_at` | Timestamps |

**Расчетные поля:** `hasNarratorCommentary` (наличие `<>` в тексте), `isGenderSpecific` (гендерное деление в шаблоне)

---

## 2. `QUESTS` (Квесты)

| Поле | Описание |
|:---|:---|
| `id` | PK |
| `MinLevel`, `MaxLevel` | Диапазон уровней |
| `name_en` | Название (референс) |
| `ZoneOrSort` | Зона/тип, опционально |
| `created_at`, `updated_at` | Timestamps |

---

## 3. `SPEAKERS` (Говорители)

| Поле | Описание |
|:---|:---|
| `id` | PK |
| `name_en` | Имя (референс) |
| `race` | Раса (`human`, `dwarf`, ...) |
| `gender` | Пол (`male`, `female`) |
| `NPCSoundID` | Идентификатор для маппинга голоса |
| `specific_voice_id` | Ссылка на `VOICES`, опционально |
| `created_at`, `updated_at` | Timestamps |

---

## 4. `VOICES` (Голоса и настройки)

| Поле | Описание |
|:---|:---|
| `id` | PK |
| `reference_file` | Файл-референс для AI-клонирования |
| `locale` | Язык (`ruRU`, `enGB`, ...) |
| `name` | Название (шаблон `{race}_{gender}_{NPCSoundID}` или кастомное) |
| `created_at`, `updated_at` | Timestamps |

---

## 5. `GENERATIONS` (Попытки генерации)

| Поле | Описание |
|:---|:---|
| `id` | PK |
| `target_gender` | `male` / `female` / `null` |
| `replicaID` | Ссылка на `REPLICS` |
| `approve_status` | Подтверждение/отклонение (опционально) |
| `text` | Очищенный/шаблонизированный текст (после этапа `text_templated`) |
| `generated_file` | Ссылка на итоговый файл в S3 (после `merged`) |
| `status` | `initial` → `text_templated` → `accented` → `validated` → `splitted` → `merged` / `error` |
| `specific_voice_id` | Ссылка на `VOICES`, опционально |
| `is_active` | Флаг актуальности (`true` только один на `replicaID` + `target_gender`) |
| `locale` | Язык генерации (`ruRU`, `enGB`, ...) |
| `isGenderSpecific` | `true` если требуются отдельные генерации для male/female |
| `created_at`, `updated_at`, `history` | Timestamps + история переходов/лог ошибок (jsonb) |

---

## 6. `GENERATION_BATCHES` (Фрагменты генерации)

**Описание:** Фрагменты текста для поэтапной генерации. Создаются на этапе `splitted`.

| Поле | Описание |
|:---|:---|
| `id` | PK |
| `generation_id` | FK → `GENERATIONS.id` |
| `voice_id` | FK → `VOICES.id` (голос для этого батча) |
| `text` | Фрагмент текста для озвучки |
| `order_index` | Integer — порядок склейки (0, 1, 2, ...) |
| `status` | Enum: `pending` → `complete` / `error` |
| `generated_file` | String — S3 URL аудиофайла в формате OGG (после генерации) |
| `created_at`, `updated_at` | Timestamps |

**Связи:**
- Один `GENERATION` → много `GENERATION_BATCHES`
- Все батчи должны иметь `status='complete'` для перехода GENERATION в `merged`

---

## 🔗 Связанные документы

- [`BACKEND_GENERATION_STAGES.md`](./BACKEND_GENERATION_STAGES.md) — Этап `splitted` (создание батчей)
- [`BACKEND_API_REFERENCE.md`](./BACKEND_API_REFERENCE.md) — API endpoints для работы с батчами
- [`BACKEND_WORKER_ARCHITECTURE.md`](./BACKEND_WORKER_ARCHITECTURE.md) — Синтез батчей (воркер)
