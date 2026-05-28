# Лабораторная работа №1


## Тема

Разработка сервиса для управления личными финансами.

## Ссылки

* Репозиторий с кодом лабораторной работы: **[ЛР1](https://github.com/FDanilka/ITMO_ICT_WebDevelopment_tools_2025-2026/tree/main/students/k3340/Fedorov_Daniil/LR1)**
* Практическая работа 1: **[Практика 1](https://github.com/FDanilka/ITMO_ICT_WebDevelopment_tools_2025-2026/tree/main/students/k3340/Fedorov_Daniil/PRs/PR1/Practice_1)**
* Практическая работа 2: **[Практика 2](https://github.com/FDanilka/ITMO_ICT_WebDevelopment_tools_2025-2026/tree/main/students/k3340/Fedorov_Daniil/PRs/PR2/Practice_2)**
* Практическая работа 3: **[Практика 3](https://github.com/FDanilka/ITMO_ICT_WebDevelopment_tools_2025-2026/tree/main/students/k3340/Fedorov_Daniil/PRs/PR3/Practice_3)**

## Описание предметной области

В реализованном приложении пользователь может зарегистрироваться, войти в систему, создать категории доходов и расходов, добавить финансовые операции, прикрепить к ним теги, создать бюджет на месяц и получить отчёты по своим финансам. Также реализована проверка превышения бюджета, которая выступает в роли простого уведомления внутри API.

## Используемые технологии

В проекте использованы следующие технологии:

* **Python** основной язык программирования;
* **FastAPI** фреймворк для создания REST API;
* **SQLModel** ORM для описания моделей данных и работы с базой;
* **PostgreSQL** реляционная база данных;
* **Alembic** инструмент для создания и применения миграций;
* **python-dotenv** загрузка переменных окружения из `.env`;
* **passlib** хэширование паролей;
* **python-jose** создание и проверка JWT-токенов;
* **Uvicorn** ASGI сервер для запуска FastAPI-приложения.

## Структура проекта

Основная логика разделена по файлам. В файле `main.py` создаётся приложение FastAPI и подключаются роутеры. В `connection.py` находится подключение к базе данных. В `models.py` описаны таблицы и схемы данных. В папке `routers` находятся отдельные файлы с endpoint-ами для разных частей приложения.

## Подключение к базе данных

Для подключения к базе данных используется файл `connection.py`. Строка подключения хранится в `.env`, чтобы не хранить пароль от базы данных прямо в коде.

```python
import os

from dotenv import load_dotenv
from sqlmodel import Session, SQLModel, create_engine


load_dotenv()

db_url = os.getenv("DB_ADMIN")

engine = create_engine(db_url, echo=True)


def init_db():
    SQLModel.metadata.create_all(engine)


def get_session():
    with Session(engine) as session:
        yield session
```

Функция `get_session()` используется в endpoint-ах через `Depends`.

В файле `.env` были добавлены переменные:

```env
DB_ADMIN=postgresql://postgres:password@localhost/finance_db
SECRET_KEY=finance_secret_key
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=60
```

Файл `.env` добавлен в `.gitignore`, так как он содержит данные подключения и секретный ключ.

## Модель данных

В проекте реализовано 7 таблиц:

* `users`  пользователи;
* `categories`  категории доходов и расходов;
* `transactions`  финансовые операции;
* `budgets`  бюджеты по категориям;
* `tags`  теги для операций;
* `transaction_tag_links`  ассоциативная таблица для связи операций и тегов;
* `goals`  финансовые цели пользователя.

### Таблица `users`

Таблица `users` хранит информацию о пользователях системы.

Основные поля:

* `id`  идентификатор пользователя;
* `username`  имя пользователя;
* `email`  электронная почта;
* `hashed_password`  хэшированный пароль.

Обычный пароль в базе данных не хранится. При регистрации пароль хэшируется, и в таблицу записывается только результат хэширования

```python
class User(SQLModel, table=True):
    __tablename__ = "users"
    id: int | None = Field(default=None, primary_key=True)
    username: str = Field(index=True, max_length=50)
    email: str = Field(index=True, unique=True, max_length=100)
    hashed_password: str

    transactions: list["Transaction"] = Relationship(back_populates="user")
    budgets: list["Budget"] = Relationship(back_populates="user")
    goals: list["Goal"] = Relationship(back_populates="user")

class UserCreate(SQLModel):
    username: str
    email: EmailStr
    password: str


class UserRead(SQLModel):
    id: int
    username: str
    email: str


class UserLogin(SQLModel):
    email: EmailStr
    password: str


class UserChangePassword(SQLModel):
    old_password: str
    new_password: str
```

### Таблица `categories`

Таблица `categories` хранит категории финансовых операций.

Основные поля:

* `id`  идентификатор категории;
* `name`  название категории;
* `type`  тип категории: `income` или `expense`.

Примеры категорий: `Food`, `Transport`, `Salary`.

```python
class CategoryBase(SQLModel):
    name: str = Field(max_length=100)
    type: str = Field(max_length=100)

class Category(CategoryBase, table=True):
    __tablename__ = "categories"

    id: int | None = Field(default=None, primary_key=True)

    transactions: list["Transaction"] = Relationship(back_populates="category")
    budgets: list["Budget"] = Relationship(back_populates="category")



class CategoryCreate(CategoryBase):
    pass

class CategoryRead(CategoryBase):
    id: int

class CategoryUpdate(SQLModel):
    name: str | None = None
    type: str | None = None
```

### Таблица `transactions`

Таблица `transactions` хранит доходы и расходы пользователя.

Основные поля:

* `id`  идентификатор операции;
* `amount`  сумма операции;
* `type`  тип операции: `income` или `expense`;
* `description`  описание операции;
* `date`  дата операции;
* `user_id`  ссылка на пользователя;
* `category_id`  ссылка на категорию.

При создании операции `user_id` не передаётся вручную. Он берётся из JWT-токена текущего пользователя.

```python
class TransactionBase(SQLModel):
    amount: float = Field(gt=0)
    type: str = Field(max_length=20)
    description: str | None = Field(default=None, max_length=255)
    date: Date = Field(default_factory=Date.today)
    category_id: int = Field(foreign_key="categories.id")


class Transaction(TransactionBase, table=True):
    __tablename__ = "transactions"

    id: int | None = Field(default=None, primary_key=True)

    user_id: int = Field(foreign_key="users.id")

    user: User | None = Relationship(back_populates="transactions")
    category: Category | None = Relationship(back_populates="transactions")
    tags: list["Tag"] = Relationship(back_populates="transactions", link_model=TransactionTagLink)


class TransactionCreate(TransactionBase):
    pass

class TransactionRead(TransactionBase):
    id: int
    user_id: int
    category: CategoryRead | None = None
    tags: list["TagRead"] = Field(default_factory=list)

class TransactionUpdate(SQLModel):
    amount: float | None = Field(default=None, gt=0)
    type: str | None = None
    description: str | None = None
    date: Date | None = None
    category_id: int | None = None
```

### Таблица `budgets`

Таблица `budgets` хранит бюджеты пользователя по категориям.

Основные поля:

* `id`  идентификатор бюджета;
* `amount_limit`  лимит расходов;
* `month`  месяц в формате `YYYY-MM`;
* `user_id`  ссылка на пользователя;
* `category_id`  ссылка на категорию.

Бюджет можно создать только для категории с типом `expense`.

```python
class BudgetBase(SQLModel):
    amount_limit: float = Field(gt=0)
    month: str = Field(max_length=10)
    category_id: int = Field(foreign_key="categories.id")

class Budget(BudgetBase, table=True):
    __tablename__ = "budgets"
    id: int | None = Field(default=None, primary_key=True)
    user_id: int = Field(foreign_key="users.id")

    user: User | None = Relationship(back_populates="budgets")

    category: Category | None = Relationship(back_populates="budgets")


class BudgetCreate(BudgetBase):
    pass

class BudgetRead(BudgetBase):
    id: int
    user_id: int
    category: CategoryRead | None = None

class BudgetUpdate(SQLModel):
    amount_limit: float | None = Field(default=None, gt=0)
    month: str | None = None
    category_id: int | None = None
```

### Таблица `tags`

Таблица `tags` хранит теги, которые можно добавлять к финансовым операциям.

Основные поля:

* `id`  идентификатор тега;
* `name`  название тега.

Примеры тегов: `card`, `cash`, `important`.

```python
class TagBase(SQLModel):
    name: str = Field(max_length=100)


class Tag(TagBase, table=True):
    __tablename__ = "tags"

    id: int | None = Field(default=None, primary_key=True)

    transactions: list["Transaction"] = Relationship(back_populates="tags", link_model=TransactionTagLink )

class TagCreate(TagBase):
    pass

class TagRead(TagBase):
    id: int

class TagUpdate(SQLModel):
    name: str | None = None
```

### Таблица `transaction_tag_links`

Таблица `transaction_tag_links` реализует связь many-to-many между операциями и тегами.

Основные поля:

* `transaction_id`  ссылка на операцию;
* `tag_id`  ссылка на тег;
* `comment`  дополнительное поле связи.

Поле `comment` показывает, что ассоциативная таблица содержит не только два внешних ключа, но и дополнительную информацию о связи. Например, при добавлении тега к операции можно указать комментарий `Paid by card`.

```python
class TransactionTagLink(SQLModel, table=True):
    __tablename__ = "transaction_tag_links"

    transaction_id: int | None = Field(default=None, foreign_key="transactions.id", primary_key=True)
    tag_id: int | None = Field(default=None, foreign_key="tags.id", primary_key=True)
    comment: str | None = Field(default=None, max_length=255)
```

### Таблица `goals`

Таблица `goals` хранит финансовые цели пользователя.

Основные поля:

* `id`  идентификатор цели;
* `title`  название цели;
* `target_amount`  необходимая сумма;
* `current_amount`  текущая накопленная сумма;
* `deadline`  срок достижения цели;
* `user_id`  ссылка на пользователя.

```python
class GoalBase(SQLModel):
    title: str = Field(max_length=100)
    target_amount: float = Field(gt=0)
    current_amount: float = Field(default=0, ge=0)
    deadline: Date

class Goal(GoalBase, table=True):
    __tablename__ = "goals"

    id: int | None = Field(default=None, primary_key=True)
    user_id: int = Field(foreign_key="users.id")

    user: User | None = Relationship(back_populates="goals")

class GoalCreate(GoalBase):
    pass

class GoalRead(GoalBase):
    id: int
    user_id: int

class GoalUpdate(SQLModel):
    title: str | None = None
    target_amount: float | None = Field(default=None, gt=0)
    current_amount: float | None = Field(default=None, ge=0)
    deadline: Date | None = None
```

## Связи между таблицами

В модели данных реализованы связи one-to-many и many-to-many.

Связи one-to-many:

* один пользователь может иметь много финансовых операций;
* один пользователь может иметь много бюджетов;
* один пользователь может иметь много финансовых целей;
* одна категория может использоваться во многих операциях;
* одна категория может использоваться во многих бюджетах.

Связь many-to-many:

* одна операция может иметь несколько тегов;
* один тег может быть привязан к нескольким операциям.

Эта связь реализована через таблицу `transaction_tag_links`, в которой есть дополнительное поле `comment`.

## Миграции базы данных

Для управления изменениями структуры базы данных используется Alembic. После описания моделей была создана первая миграция:

```bash
alembic revision --autogenerate -m "create finance tables"
```

Затем миграция была применена к базе данных:

```bash
alembic upgrade head
```

В результате в PostgreSQL были созданы таблицы приложения.

## Авторизация и пользователи

В проекте реализована регистрация и авторизация пользователей.

Основные endpoint-ы:

* `POST /auth/register`  регистрация пользователя;
* `POST /auth/login`  вход пользователя и получение JWT-токена;
* `GET /users/me`  получение текущего пользователя;
* `GET /users/`  получение списка пользователей;
* `PATCH /users/change-password`  смена пароля.

При регистрации пользователь передаёт имя, email и пароль. Пароль хэшируется с помощью `passlib`, после чего в базу данных сохраняется только хэш.

После успешного входа пользователь получает JWT-токен. Этот токен используется для доступа к защищённым endpoint-ам. Если токен отсутствует или неверный, API возвращает ошибку авторизации.

## Реализованные endpoint-ы

### Auth

| Метод | Endpoint         | Назначение               |
| ----- | ---------------- | ------------------------ |
| POST  | `/auth/register` | Регистрация пользователя |
| POST  | `/auth/login`    | Авторизация пользователя |

### Users

| Метод | Endpoint                 | Назначение                      |
| ----- | ------------------------ | ------------------------------- |
| GET   | `/users/me`              | Получение текущего пользователя |
| GET   | `/users/`                | Получение списка пользователей  |
| PATCH | `/users/change-password` | Смена пароля                    |

### Categories

| Метод  | Endpoint                    | Назначение                 |
| ------ | --------------------------- | -------------------------- |
| POST   | `/categories/`              | Создание категории         |
| GET    | `/categories/`              | Получение списка категорий |
| GET    | `/categories/{category_id}` | Получение одной категории  |
| PATCH  | `/categories/{category_id}` | Обновление категории       |
| DELETE | `/categories/{category_id}` | Удаление категории         |

Для категорий реализована проверка удаления: если категория уже используется в операциях или бюджетах, удалить её нельзя.

### Tags

| Метод  | Endpoint         | Назначение             |
| ------ | ---------------- | ---------------------- |
| POST   | `/tags/`         | Создание тега          |
| GET    | `/tags/`         | Получение списка тегов |
| GET    | `/tags/{tag_id}` | Получение одного тега  |
| PATCH  | `/tags/{tag_id}` | Обновление тега        |
| DELETE | `/tags/{tag_id}` | Удаление тега          |

При удалении тега удаляются связи этого тега с операциями, но сами операции не удаляются.

### Transactions

| Метод  | Endpoint                                       | Назначение                             |
| ------ | ---------------------------------------------- | -------------------------------------- |
| POST   | `/transactions/`                               | Создание операции                      |
| GET    | `/transactions/`                               | Получение списка операций пользователя |
| GET    | `/transactions/{transaction_id}`               | Получение одной операции               |
| PATCH  | `/transactions/{transaction_id}`               | Обновление операции                    |
| DELETE | `/transactions/{transaction_id}`               | Удаление операции                      |
| POST   | `/transactions/{transaction_id}/tags`          | Добавление тега к операции             |
| DELETE | `/transactions/{transaction_id}/tags/{tag_id}` | Удаление тега у операции               |

При создании операции проверяется, что категория существует и что тип операции совпадает с типом категории. Например, операция с типом `expense` не может быть создана в категории `Salary`, если эта категория имеет тип `income`.

При удалении операции удаляются связи с тегами, но сами теги остаются в системе.

### Budgets

| Метод  | Endpoint               | Назначение                             |
| ------ | ---------------------- | -------------------------------------- |
| POST   | `/budgets/`            | Создание бюджета                       |
| GET    | `/budgets/`            | Получение списка бюджетов пользователя |
| GET    | `/budgets/{budget_id}` | Получение одного бюджета               |
| PATCH  | `/budgets/{budget_id}` | Обновление бюджета                     |
| DELETE | `/budgets/{budget_id}` | Удаление бюджета                       |

Бюджет можно создать только для категории расходов.

### Goals

| Метод  | Endpoint           | Назначение                          |
| ------ | ------------------ | ----------------------------------- |
| POST   | `/goals/`          | Создание финансовой цели            |
| GET    | `/goals/`          | Получение списка целей пользователя |
| GET    | `/goals/{goal_id}` | Получение одной цели                |
| PATCH  | `/goals/{goal_id}` | Обновление цели                     |
| DELETE | `/goals/{goal_id}` | Удаление цели                       |

Цели принадлежат конкретному пользователю. Пользователь может просматривать и изменять только свои цели.

### Reports

| Метод | Endpoint                 | Назначение                    |
| ----- | ------------------------ | ----------------------------- |
| GET   | `/reports/summary`       | Общий финансовый отчёт        |
| GET   | `/reports/by-category`   | Анализ расходов по категориям |
| GET   | `/reports/budget-status` | Проверка превышения бюджета   |

Отчёты формируются только по данным текущего пользователя.



## Вывод

В ходе лабораторной работы был разработан REST API сервис для управления личными финансами. Приложение позволяет пользователю регистрироваться, авторизоваться, добавлять доходы и расходы, создавать категории, бюджеты, теги и финансовые цели, а также получать отчёты по своим финансам.

Модель данных соответствует требованиям лабораторной работы: в ней больше пяти таблиц, есть связи one-to-many и many-to-many, а ассоциативная таблица содержит дополнительное поле. Также в проекте реализованы миграции через Alembic и пользовательская авторизация с использованием JWT-токенов.
