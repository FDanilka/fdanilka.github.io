# Лабораторная работа №1

## Тема

Разработка сервиса для управления личными финансами.

## Цель работы

Разработать серверное приложение на FastAPI с использованием PostgreSQL, SQLModel, Alembic и JWT-авторизации.

## Ссылки

- Репозиторий с кодом лабораторной: ВСТАВИТЬ_ССЫЛКУ
- Практическая работа 1: ВСТАВИТЬ_ССЫЛКУ
- Практическая работа 2: ВСТАВИТЬ_ССЫЛКУ
- Практическая работа 3: ВСТАВИТЬ_ССЫЛКУ

## Описание проекта

Сервис предназначен для управления личными финансами. Пользователь может добавлять доходы и расходы, создавать категории, устанавливать бюджеты, добавлять теги к операциям, формировать финансовые отчёты и создавать финансовые цели.

## Используемые технологии

- Python
- FastAPI
- SQLModel
- PostgreSQL
- Alembic
- JWT
- Passlib
- python-jose

## Модель данных

В проекте реализованы таблицы:

- users
- categories
- transactions
- budgets
- tags
- transaction_tag_links
- goals

## Связи между таблицами

Связи one-to-many:

- User → Transaction
- User → Budget
- User → Goal
- Category → Transaction
- Category → Budget

Связь many-to-many:

- Transaction ↔ Tag

Связь реализована через таблицу transaction_tag_links. В ней есть дополнительное поле comment.

## Основные эндпоинты

### Auth

- POST /auth/register
- POST /auth/login

### Users

- GET /users/me
- GET /users/
- PATCH /users/change-password

### Categories

- POST /categories/
- GET /categories/
- GET /categories/{category_id}
- PATCH /categories/{category_id}
- DELETE /categories/{category_id}

### Transactions

- POST /transactions/
- GET /transactions/
- GET /transactions/{transaction_id}
- PATCH /transactions/{transaction_id}
- DELETE /transactions/{transaction_id}
- POST /transactions/{transaction_id}/tags
- DELETE /transactions/{transaction_id}/tags/{tag_id}

### Budgets

- POST /budgets/
- GET /budgets/
- GET /budgets/{budget_id}
- PATCH /budgets/{budget_id}
- DELETE /budgets/{budget_id}

### Goals

- POST /goals/
- GET /goals/
- GET /goals/{goal_id}
- PATCH /goals/{goal_id}
- DELETE /goals/{goal_id}

### Reports

- GET /reports/summary
- GET /reports/by-category
- GET /reports/budget-status

## Примеры работы API

### Регистрация пользователя


### Авторизация пользователя


### Создание транзакции


### Отчёт по бюджету


## Вывод

В ходе лабораторной работы был разработан REST API сервис для управления личными финансами. В приложении реализованы пользователи, JWT-авторизация, доходы и расходы, категории, бюджеты, теги, финансовые цели и отчёты. Модель данных содержит связи one-to-many и many-to-many, а ассоциативная таблица имеет дополнительное поле.