# Техническое задание: Backend «Видео Помощник» (MVP)

## 1. Технологии
- **Язык:** Python 3.10+
- **Фреймворк:** FastAPI (авто-документация Swagger)
- **ORM:** SQLAlchemy + Alembic
- **База данных:** PostgreSQL (разработку можно начать с SQLite)
- **Хранение файлов:** локальная папка `uploads/` (позже S3)
- **AI-сервис:** DeepSeek API (ключ `sk-a4e6c235b1c84d2aa8944b6cba6ed52b`)

## 2. Структура проекта
backend/
├── app/
│ ├── main.py # запуск FastAPI
│ ├── models.py # SQLAlchemy-модели
│ ├── schemas.py # Pydantic-схемы
│ ├── database.py # подключение к БД
│ ├── routers/
│ │ ├── auth.py # регистрация/логин (заглушка)
│ │ ├── projects.py # CRUD проектов
│ │ ├── equipment.py # загрузка CSV и каталог
│ │ └── audit.py # AI-аудит (DeepSeek)
│ └── services/
│ ├── csv_parser.py
│ └── ai_auditor.py
├── requirements.txt
└── .env

text

## 3. Модели данных (ключевые)

### User
- `id`: int, pk
- `email`: str, unique
- `password_hash`: str

### Project
- `id`: int, pk
- `user_id`: FK → User
- `name`: str
- `plan_image_url`: str (путь к плану)
- `walls`: JSON (массив стен)
- `devices`: JSON (массив устройств, см. ниже)
- `created_at`: datetime

### Equipment (справочник)
- `article`: str, pk
- `name`: str
- `category`: str (camera, recorder, switch)
- `angle_h`: float (горизонтальный угол обзора)
- `angle_v`: float (опционально)
- `resolution`: str (1920×1080)
- `mount`: str (wall, ceiling)
- `price`: float

### Device (внутри JSON)
```json
{
  "id": "uuid",
  "type": "camera",
  "article": "CAM-123",
  "x": 150.0,
  "y": 200.0,
  "angle": 45,
  "mount": "wall",
  "count": 1
}
4. API-эндпоинты (MVP)
Загрузка прайс-листа

POST /api/equipment/upload (multipart/form-data, файл CSV)

CSV-колонки: article, name, category, angle_h, angle_v, resolution, mount, price

Парсинг, валидация, сохранение в Equipment (перезапись по article).

Ответ: количество записей.

Каталог оборудования

GET /api/equipment?category=camera&search=...

Проекты

POST /api/projects — создать

GET /api/projects — список

GET /api/projects/{id} — детали

PUT /api/projects/{id} — обновить (walls, devices)

DELETE /api/projects/{id}

AI-аудит (самое важное!)

POST /api/projects/{id}/audit

Логика:

Собрать все камеры проекта, для каждой подтянуть angle_h, resolution из Equipment.

Сформировать описание сцены (стены, камеры) в JSON.

Отправить запрос в DeepSeek API (https://api.deepseek.com/v1/chat/completions) с промптом, требующим найти слепые зоны, проверить плотность пикселей и дать рекомендации.

Ответ ИИ вернуть фронтенду.

Модель: deepseek-chat

Аутентификация: Authorization: Bearer sk-a4e6c235b1c84d2aa8944b6cba6ed52b

5. С чего начать (пошагово)
Настроить FastAPI, SQLAlchemy, uvicorn.

Реализовать модель Equipment и CSV-парсер.

Сделать эндпоинт загрузки CSV и GET-каталог.

Реализовать CRUD проектов (упрощённо, без авторизации).

Разработать AI-аудит: написать ai_auditor.py, который формирует промпт и общается с DeepSeek.

Проверить всё через Swagger (/docs).

6. Что будет дальше
Авторизация (JWT) и привязка проектов к пользователям.

Docker-образы, CI/CD.

Интеграция с React-фронтендом.

Свяжись с основателем, когда будут вопросы. Удачи!
