# VoiceOverCraft — AI-озвучивание квестов WoW

## 🎯 Назначение

Backend для генерации озвучки квестов и фраз в World of Warcraft через AI.

**Критичные ограничения:**
- 🪟 **Воркер работает на Windows** (GPU/CPU-intensive задачи)
- 📦 **Формат аудио:** OGG, загрузка через S3
- 🚫 **Нет блокировок в БД** — polling через REST API (нет `worker_id`/`locked_at`)
- ⚡ **is_active флаг** — одна активная генерация на `replicaID` + `target_gender`

---

## 🏗 Архитектура (5 компонентов)

1. **Backend** — FastAPI + Alembic + PostgreSQL (данные, роутинг задач для воркеров/фронтенда)
2. **Хранилище файлов** — Аудио-архивы и промежуточные данные (S3)
3. **Frontend** — Прослушивание результатов, мониторинг статусов генераций
4. **Воркер (Windows)** — GPU/CPU-intensive задачи, поэтапная генерация
5. **Аддон WoW** — Работа с архивами озвучки, взаимодействие с API для оценки/голосования

---

## 🛠 Технологический стек

**Backend:**
- Framework: FastAPI
- Database Migrations: Alembic
- Database: PostgreSQL

**Infrastructure:**
- Deployment: Ansible
- File Storage: S3-совместимое хранилище

**Worker:**
- Platform: Windows
- Workload: GPU/CPU-intensive задачи генерации

---

## 🔄 Рабочий процесс

1. В БД загружаются голоса, референсы для AI-клонирования и реплики
2. Система определяет реплики без генерации и создаёт задачу
3. Воркер берёт задачу в очередь и выполняет этапы генерации
4. По завершении формируется архив озвучек (реплики с `is_active=true` или с самой свежей `updated_at`)
5. Игрок подключает архив как аддон
6. При низком качестве игрок ставит дизлайк (аддон/фронтенд) или загружает другой голос для перегенерации
7. Новая озвучка попадает в следующую выгрузку, цикл повторяется

**Наполнение и деплой:**
- v1: Выгрузка текстов из `cmangos` (Vanilla)
- Далее: Выгрузка из других ядер приватных WoW
- Первичное наполнение БД → скрипты
- Развертывание → Ansible

---

## 📚 Детальная документация

### Backend (`docs/backend/`)

| Файл | Зачем идти |
|------|------------|
| [`BACKEND_GENERATION_STAGES.md`](docs/backend/BACKEND_GENERATION_STAGES.md) | 6 этапов генерации + примеры воркеров |
| [`BACKEND_DATA_MODELS.md`](docs/backend/BACKEND_DATA_MODELS.md) | 6 таблиц БД (REPLICS, QUESTS, SPEAKERS, VOICES, GENERATIONS, GENERATION_BATCHES) |
| [`BACKEND_QUEUE_SYSTEM.md`](docs/backend/BACKEND_QUEUE_SYSTEM.md) | REST API для очередей (3 endpoint, polling) |
| [`BACKEND_VOICE_LOGIC.md`](docs/backend/BACKEND_VOICE_LOGIC.md) | Приоритет выбора голосов + дикторские блоки |
| [`BACKEND_DEVELOPMENT_STAGES.md`](docs/backend/BACKEND_DEVELOPMENT_STAGES.md) | Этапы разработки |
| [`BACKEND_OPEN_QUESTIONS.md`](docs/backend/BACKEND_OPEN_QUESTIONS.md) | Открытые вопросы для обсуждения |

### Frontend / Worker
_Документация в разработке_

---

## 🚀 Типовые задачи агента

**Модели данных:** → [`BACKEND_DATA_MODELS.md`](docs/backend/BACKEND_DATA_MODELS.md)  
**Воркеры:** → [`BACKEND_GENERATION_STAGES.md`](docs/backend/BACKEND_GENERATION_STAGES.md)  
**API endpoints:** → [`BACKEND_QUEUE_SYSTEM.md`](docs/backend/BACKEND_QUEUE_SYSTEM.md)  
**Голоса:** → [`BACKEND_VOICE_LOGIC.md`](docs/backend/BACKEND_VOICE_LOGIC.md)  
**Наполнение БД:** → [`HOW_TO_FILL_DATABASE.md`](HOW_TO_FILL_DATABASE.md)
