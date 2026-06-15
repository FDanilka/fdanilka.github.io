# Лабораторная работа 3

# Упаковка FastAPI приложения в Docker, работа с источниками данных и очереди

## Цель работы

Научиться упаковывать FastAPI приложение в Docker, интегрировать парсер данных с базой данных и вызывать парсер через API и очередь.


# 1. Подзадача 1. Упаковка FastAPI-приложения, базы данных и парсера в Docker


## 1.1. Использование существующего парсера

Парсер из лабораторной работы 2 уже находился в папке:

```text
task2/
```

Поэтому новый парсер не создавался с нуля. В файл `task2/parser.py` была добавлена общая функция:

```python
def parse_url_sync(url: str, approach: str = "http_parser") -> dict:
    response = requests.get(url, headers=HEADERS, timeout=10)
    response.raise_for_status()

    title = extract_title(response.text)

    save_page_sync(url, title, approach)

    return {
        "url": url,
        "title": title,
        "approach": approach,
        "message": "Parsing completed"
    }
```

Эта функция выполняет полный цикл обработки одного URL:

1. получает адрес страницы;
2. загружает HTML;
3. извлекает заголовок страницы;
4. сохраняет результат в таблицу `parsed_pages`;
5. возвращает результат в виде словаря.

Таким образом, код парсера стал переиспользуемым. Его можно вызывать:

* из отдельного parser-service;
* из Celery worker;
* из других частей приложения.


## 1.2. Таблица для результатов парсинга

Для хранения результатов парсинга используется таблица:

```text
parsed_pages
```

Модель находится в `app/models.py`:

```python
class ParsedPageBase(SQLModel):
    url: str = Field(max_length=500)
    title: str = Field(max_length=500)
    approach: str = Field(max_length=50)
    parsed_at: datetime = Field(default_factory=datetime.utcnow)


class ParsedPage(ParsedPageBase, table=True):
    __tablename__ = "parsed_pages"

    id: int | None = Field(default=None, primary_key=True)
```


## 1.3. Создание отдельного FastAPI-сервиса парсера

Для выполнения требования о вызове парсера по HTTP был создан отдельный сервис:

```text
parser_service/main.py
```

Код сервиса:

```python
import requests
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

from task2.parser import parse_url_sync


app = FastAPI(title="Parser Service")


class ParseRequest(BaseModel):
    url: str


class ParseResponse(BaseModel):
    url: str
    title: str
    approach: str
    message: str


@app.get("/")
def root():
    return {"message": "Parser service is working"}


@app.post("/parse", response_model=ParseResponse)
def parse(request: ParseRequest):
    try:
        result = parse_url_sync(
            url=request.url,
            approach="http_parser_service"
        )

        return result

    except requests.RequestException as error:
        raise HTTPException(
            status_code=500,
            detail=f"Ошибка при загрузке страницы: {error}"
        )

    except Exception as error:
        raise HTTPException(
            status_code=500,
            detail=f"Ошибка при выполнении парсинга: {error}"
        )
```

Parser-service имеет endpoint:

```http
POST /parse
```

Он принимает JSON:

```json
{
  "url": "https://example.com"
}
```

И возвращает результат:

```json
{
  "url": "https://example.com",
  "title": "Example Domain",
  "approach": "http_parser_service",
  "message": "Parsing completed"
}
```



## 1.4. Создание Dockerfile

Для упаковки приложения был создан файл:

```text
Dockerfile
```

Содержимое:

```dockerfile
FROM python:3.14-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Пояснение:

| Строка                    | Назначение                                   |
| ------------------------- | -------------------------------------------- |
| `FROM python:3.14-slim`   | базовый Python-образ                         |
| `WORKDIR /app`            | рабочая директория внутри контейнера         |
| `COPY requirements.txt .` | копирование зависимостей                     |
| `RUN pip install ...`     | установка зависимостей                       |
| `COPY . .`                | копирование исходного кода                   |
| `CMD ...`                 | команда запуска по умолчанию                 |


## 1.5. Docker Compose

Для запуска всех сервисов был создан файл:

```text
docker-compose.yml
```

В нём описаны сервисы:

* `db`;
* `api`;
* `parser`;
* `redis`;
* `celery_worker`.

Фрагмент `docker-compose.yml`:

```yaml
services:
  db:
    image: postgres:16
    container_name: finance_db
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: finance_db
    ports:
      - "5434:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -U postgres -d finance_db" ]
      interval: 5s
      timeout: 5s
      retries: 5


  redis:
    image: redis:7
    container_name: finance_redis
    ports:
      - "6379:6379"

  api:
    build: .
    container_name: finance_api
    command: >
      sh -c "alembic upgrade head && uvicorn app.main:app --host 0.0.0.0 --port 8000"
    environment:
      DB_ADMIN: postgresql://postgres:password@db:5432/finance_db
      DB_ADMIN_ASYNC: postgresql+asyncpg://postgres:password@db:5432/finance_db
      SECRET_KEY: finance_secret_key
      ALGORITHM: HS256
      ACCESS_TOKEN_EXPIRE_MINUTES: 60
      PARSER_SERVICE_URL: http://parser:8001
      CELERY_BROKER_URL: redis://redis:6379/0
      CELERY_RESULT_BACKEND: redis://redis:6379/1
    ports:
      - "8000:8000"
    depends_on:
      db:
        condition: service_healthy
      parser:
        condition: service_started
      redis:
        condition: service_started

  parser:
    build: .
    container_name: finance_parser
    command: uvicorn parser_service.main:app --host 0.0.0.0 --port 8001
    environment:
      DB_ADMIN: postgresql://postgres:password@db:5432/finance_db
      DB_ADMIN_ASYNC: postgresql+asyncpg://postgres:password@db:5432/finance_db
    ports:
      - "8001:8001"
    depends_on:
      db:
        condition: service_healthy

  celery_worker:
    build: .
    container_name: finance_celery_worker
    command: celery -A celery_worker.celery_app:app worker --loglevel=info
    environment:
      DB_ADMIN: postgresql://postgres:password@db:5432/finance_db
      DB_ADMIN_ASYNC: postgresql+asyncpg://postgres:password@db:5432/finance_db
      CELERY_BROKER_URL: redis://redis:6379/0
      CELERY_RESULT_BACKEND: redis://redis:6379/1
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

volumes:
  postgres_data:
```


# 2. Подзадача 2. Вызов парсера из FastAPI

## 2.1. Добавление router в основное приложение

Для вызова parser-service из основного FastAPI-приложения был создан файл:

```text
app/routers/parser.py
```

Код endpoint-а:

```python
load_dotenv()

router = APIRouter(prefix="/parser", tags=["Parser"])


PARSER_SERVICE_URL = os.getenv(
    "PARSER_SERVICE_URL",
    "http://127.0.0.1:8001"
)


class ParseRequest(BaseModel):
    url: str


@router.post("/parse")
def parse_url(request: ParseRequest):
    try:
        response = requests.post(
            f"{PARSER_SERVICE_URL}/parse",
            json={"url": request.url},
            timeout=15
        )

        response.raise_for_status()

        return {
            "message": "Парсер сработал",
            "parser_response": response.json()
        }

    except requests.RequestException as error:
        raise HTTPException(
            status_code=500,
            detail=f"Ошибка при вызове parser-service: {error}"
        )
```

Endpoint:

```http
POST /parser/parse
```

Он принимает:

```json
{
  "url": "https://example.com"
}
```

Основное приложение не выполняет парсинг самостоятельно. Оно отправляет HTTP-запрос в отдельный parser-service:

```text
http://parser:8001/parse
```

# 3. Подзадача 3. Вызов парсера через очередь

В этом варианте FastAPI не ждёт завершения парсинга. Он только ставит задачу в очередь и сразу возвращает клиенту `task_id`.


## 3.1. Настройка Celery

Был создан файл:

```text
celery_worker/celery_app.py
```

Код:

```python
import os

from celery import Celery


CELERY_BROKER_URL = os.getenv(
    "CELERY_BROKER_URL",
    "redis://localhost:6379/0"
)

CELERY_RESULT_BACKEND = os.getenv(
    "CELERY_RESULT_BACKEND",
    "redis://localhost:6379/1"
)


app = Celery(
    "finance_parser_worker",
    broker=CELERY_BROKER_URL,
    backend=CELERY_RESULT_BACKEND,
    include=["celery_worker.tasks"]
)

app.conf.update(
    task_track_started=True,
    result_expires=3600,
)
```

Здесь:

| Переменная                        | Назначение                                 |
| --------------------------------- | ------------------------------------------ |
| `CELERY_BROKER_URL`               | адрес Redis для постановки задач в очередь |
| `CELERY_RESULT_BACKEND`           | адрес Redis для хранения результатов задач |
| `include=["celery_worker.tasks"]` | модуль, где Celery ищет задачи             |

## 3.2. Создание Celery-задачи

Был создан файл:

```text
celery_worker/tasks.py
```

Код:

```python
from celery_worker.celery_app import app
from task2.parser import parse_url_sync


@app.task(name="parse_url_task")
def parse_url_task(url: str) -> dict:
    result = parse_url_sync(
        url=url,
        approach="celery_worker"
    )

    return result
```

Эта задача вызывает уже существующий парсер из лабораторной работы 2.

Преимущество такого подхода в том, что парсинг выполняется в фоне, а основной API остаётся отзывчивым и не блокируется на время загрузки страницы.

## 3.3. Endpoint для постановки задачи в очередь

В `app/routers/parser.py` был добавлен endpoint:

```python
@router.post("/parse-async")
def parse_url_async(request: ParseRequest):
    task = parse_url_task.delay(request.url)

    return {
        "message": "Parsing task added to queue",
        "task_id": task.id,
        "status": "PENDING"
    }
```

Endpoint:

```http
POST /parser/parse-async
```

Пример запроса:

```json
{
  "url": "https://example.com"
}
```

Пример ответа:

```json
{
  "message": "Parsing task added to queue",
  "task_id": "example-task-id",
  "status": "PENDING"
}
```

## 3.4. Endpoint для проверки статуса задачи

Также был добавлен endpoint:

```python
@router.get("/tasks/{task_id}")
def get_task_status(task_id: str):
    task_result = AsyncResult(task_id, app=celery_app)

    response = {
        "task_id": task_id,
        "status": task_result.status
    }

    if task_result.successful():
        response["result"] = task_result.result

    elif task_result.failed():
        response["error"] = str(task_result.result)

    return response
```

Endpoint:

```http
GET /parser/tasks/{task_id}
```

Он позволяет проверить состояние фоновой задачи.


# 4. Вывод

В ходе лабораторной работы было выполнено контейнеризованное развертывание FastAPI-приложения, базы данных PostgreSQL, отдельного сервиса парсера, Redis и Celery worker. В работе были способы контейнеризации, взаимодействия сервисов, фоновой обработки задач и интеграции парсера с FastAPI-приложением.
