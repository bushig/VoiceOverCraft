# VoiceOverCraft Documentation

## 📚 Структура документации

### Backend

Расположен в `docs/backend/`:

- **`BACKEND_OVERVIEW.md`** — Обзор проекта, цели и основные задачи
- **`BACKEND_ARCHITECTURE.md`** — Архитектура системы (5 компонентов)
- **`BACKEND_WORKFLOW.md`** — Рабочий процесс генерации озвучки
- **`BACKEND_DEVELOPMENT_STAGES.md`** — Этапы разработки
- **`BACKEND_TECH_STACK.md`** — Технологический стек (FastAPI, Alembic, БД, Ansible)
- **`BACKEND_DATA_MODELS.md`** — Модель данных (6 таблиц: REPLICS, QUESTS, SPEAKERS, VOICES, GENERATIONS, GENERATION_BATCHES)
- **`BACKEND_VOICE_LOGIC.md`** — Логика наследования голосов
- **`BACKEND_QUEUE_SYSTEM.md`** — Очередь задач и REST API (3 endpoint, polling без блокировок)
- **`BACKEND_GENERATION_STAGES.md`** — Этапы генерации (6 статусов: initial → text_templated → accented → validated → splitted → merged)
- **`BACKEND_OPEN_QUESTIONS.md`** — Открытые вопросы и заметки для обсуждения

### Frontend

Расположен в `docs/frontend/`:

_Документация в разработке_

### Worker

Расположен в `docs/worker/`:

_Документация в разработке_

## 🔧 Использование с AI-агентами

Каждый файл представляет собой независимый блок документации, который можно:

- Отдельно прорабатывать и дополнять
- Использовать как контекст для конкретных задач
- Обновлять по мере развития проекта

## 📝 Примеры использования

**При разработке моделей данных:**
```bash
Прочитай docs/backend/BACKEND_DATA_MODELS.md и создай SQLAlchemy модели
```

**При настройке архитектуры:**
```bash
На основе docs/backend/BACKEND_ARCHITECTURE.md и docs/backend/BACKEND_TECH_STACK.md предложи структуру проекта
```

**При разработке очереди задач:**
```bash
Прочитай docs/backend/BACKEND_QUEUE_SYSTEM.md и создай API endpoints для получения задач
```

**При реализации генерации:**
```bash
На основе docs/backend/BACKEND_GENERATION_STAGES.md реализуй воркера для статуса splitted
```

**При проработке открытых вопросов:**
```bash
Проанализируй docs/backend/BACKEND_OPEN_QUESTIONS.md и предложи решения для каждого пункта
```
