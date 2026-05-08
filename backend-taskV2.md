🐍 Техническое задание: Backend «Видео Помощник» (MVP, v1.0)
1. Общая архитектура и стек
Язык: Python 3.10+

Фреймворк: FastAPI (авто-документация Swagger по адресу /docs)

ORM: SQLAlchemy (async или sync – на старте можно sync) + Alembic для миграций

База данных: PostgreSQL (в Docker; для локальной разработки можно SQLite)

Хранение файлов: локальная папка uploads/ (планы этажей, прайс-листы)

AI-сервис: DeepSeek API (ключ будет предоставлен позже)

Контейнеризация: Docker, docker-compose (для лёгкого развёртывания)

Принцип: Микросервисная структура. Даже если пока всё в одном процессе, разделение по роутерам и сервисам должно быть чётким, чтобы в будущем легко вынести модули в отдельные контейнеры.

2. Структура проекта
text
backend/
├── app/
│   ├── main.py               # Инициализация FastAPI, подключение роутеров, CORS
│   ├── database.py           # Подключение к БД, создание сессий
│   ├── models.py             # Модели SQLAlchemy
│   ├── schemas.py            # Pydantic-схемы для API-запросов/ответов
│   ├── routers/
│   │   ├── equipment.py      # Загрузка CSV и получение каталога
│   │   ├── projects.py       # CRUD проектов
│   │   └── audit.py          # AI-аудит проекта
│   └── services/
│       ├── csv_parser.py     # Умный парсер CSV
│       ├── ai_auditor.py     # Формирование промпта и вызов DeepSeek
│       ├── calculator.py     # Расчёты (архив, PoE, подбор NVR, смета)
│       └── estimate.py       # Агрегация сметы
├── requirements.txt
├── Dockerfile
└── docker-compose.yml
3. Модели данных (ключевые)
User (MVP – минимально)
id: BigInteger, pk

email: String(255), unique, not null

password_hash: String, not null

Project
id: BigInteger, pk

user_id: FK -> User

name: String(255)

plan_image_path: String(500) (путь к загруженному плану)

walls: JSONB (массив стен: [{"x1":0,"y1":0,"x2":100,"y2":0}, ...])

devices: JSONB (массив устройств, см. формат ниже)

cable_traces: JSONB (массив трасс: [{"points":[[x1,y1],[x2,y2]], "type":"manual/auto/wall"}])

created_at: DateTime (UTC)

Device (внутри JSONB)
json
{
  "id": "uuid",
  "type": "camera",
  "article": "DH-IPC-HDW2431T-AS-S2",
  "x": 150.0,
  "y": 200.0,
  "angle": 45,
  "mount": "ceiling",
  "height": 2.8,
  "count": 1
}
Equipment (справочник)
id: Integer, pk

article: String(100), unique, not null

name: String(255)

category: String(50) (camera, recorder, switch, ups, cable, bracket, box, power_supply, monitor, other)

specs: JSONB – гибкое хранение всех характеристик. Пример:

json
{
  "resolution": "1920x1080",
  "angle_h": 99,
  "angle_v": 55,
  "mount": "ceiling",
  "power_type": "PoE",
  "power_consumption_w": 7.5,
  "price_rub": 5500,
  "connector_type": "RJ45",
  "frame_rate": 25,
  "ir_range_m": 30
}
created_at: DateTime

Зачем JSONB? Чтобы не перестраивать таблицу при добавлении новых параметров (например, «тип винтика» или «материал корпуса»). CSV-парсер будет маппить колонки прямо в specs.

4. API-эндпоинты (с деталями)
4.1 Загрузка прайс-листа
POST /api/equipment/upload

Content-Type: multipart/form-data

Поле file – CSV-файл.

Логика:

Прочитать CSV (разделитель – запятая или точка с запятой, автоопределение).
Показать пользователю превью маппинга (заголовки из файла ↔ наши поля). Для этого отдельный эндпоинт:
POST /api/equipment/preview – возвращает список колонок файла и предполагаемый маппинг.
После подтверждения пользователем (передаётся JSON с маппингом) выполняется POST /api/equipment/import – парсинг, валидация, вставка/обновление в таблице equipment.
Правила валидации: обязательны article (уникальный), name, category. Остальные поля сохраняются в specs.

Ответ: количество загруженных/обновлённых записей.

4.2 Каталог оборудования
GET /api/equipment

Query-параметры: category, search (по article, name), min_price, max_price.

Возвращает массив объектов Equipment с полным specs.

4.3 Проекты
POST /api/projects – создать проект (name, опционально plan_image). Тело JSON.

GET /api/projects – список проектов пользователя.

GET /api/projects/{id} – детали проекта (включая walls, devices, cable_traces, смету).

PUT /api/projects/{id} – обновить стены, устройства, трассы. Тело JSON.

DELETE /api/projects/{id}.

4.4 AI-аудит
POST /api/projects/{id}/audit

Алгоритм:

Извлечь все устройства типа camera из проекта.
Для каждой найти запись в Equipment, получить specs (угол обзора, разрешение, высота установки).
Сформировать JSON-описание сцены:
json
{
  "walls": [...],
  "cameras": [
    {"article": "...", "x":..., "y":..., "angle":..., "height":..., "angle_h":..., "resolution":...}
  ]
}
Отправить запрос в DeepSeek API:
URL: https://api.deepseek.com/v1/chat/completions
Аутентификация: Authorization: Bearer sk-a4e6c235b1c84d2aa8944b6cba6ed52b
Модель: deepseek-chat
Промпт: строгий шаблон, требующий найти слепые зоны, проверить плотность пикселей (DORI), дать рекомендации в формате JSON.
Вернуть ответ ИИ (текст с рекомендациями) или структурированный JSON.
В MVP достаточно возвращать текстовый вывод. Обработать возможные ошибки API (timeout, превышение токенов).

4.5 Смета проекта
GET /api/projects/{id}/estimate

Логика (реализуется в services/estimate.py):

Суммировать стоимость всех устройств (price_rub из specs их Equipment).

Для каждой камеры определить необходимость кабеля (если не Wi-Fi) и рассчитать метраж по трассам из cable_traces. Стоимость кабеля = метраж × цена из Equipment (тип cable).

Рассчитать монтажные работы: для каждой камеры фиксированная стоимость (например, 1500 руб.), для регистратора – отдельная.

Учесть стоимость дополнительных БП, разъёмов, креплений, если они добавлены в проект как устройства.

Ответ: JSON со сметой (категории, цены, итоговая сумма).

5. Сервис расчётов (calculator.py)
Реализовать как набор чистых функций (чтобы можно было тестировать без БД):

calc_disk_space(cameras: list, days: int, quality_factor: float = 1.0) -> float

Вычисляет объём архива в ГБ.

Формула: битрейт (Мбит/с) * 3600 * 24 * days / 8 / 1000 / 1000 * quality_factor.

Битрейт брать из specs камеры (если нет – оценивать по разрешению).

calc_poe_budget(cameras: list) -> tuple[float, bool]

Возвращает суммарную мощность и флаг, требуется ли PoE.

suggest_recorder(cameras: list, equipment: list) -> Optional[Equipment]

Находит NVR с числом каналов ≥ количества камер и поддержкой суммарного битрейта.

suggest_switch(cameras: list, equipment: list) -> Optional[Equipment]

По числу камер, PoE-бюджету и скорости портов подбирает коммутатор.

check_cable_length(traces: list, max_utp=100) -> list[str]

Возвращает предупреждения, если длина превышает 100 м.

6. Порядок выполнения (вехи)
Настроить проект, Docker, подключить БД.

Реализовать модели Equipment, CSV-парсер и эндпоинты загрузки/просмотра.

Сделать CRUD проектов (без авторизации, предполагая одного пользователя).

Реализовать сервис AI-аудита и подключить к эндпоинту.

Реализовать сервис расчётов и эндпоинт сметы.

Покрыть всё тестами (pytest) и проверить через Swagger.

Свяжись с основателем, когда будут вопросы, или демонстрируй промежуточные результаты. Удачи!
