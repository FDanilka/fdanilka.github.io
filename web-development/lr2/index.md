# Лабораторная работа 2. Потоки. Процессы. Асинхронность.

# 1. Цель работы

Цель работы: понять отличия потоками и процессами и понять, что такое ассинхронность в Python.


# 2. Задача 1. Подсчёт суммы чисел

## 2.1. Постановка задачи

Необходимо написать три программы на Python, которые считают сумму чисел от 1 до:

```python
10_000_000_000_000
```

Для этого используются три подхода:

* `threading`;
* `multiprocessing`;
* `asyncio`.

Каждая программа содержит функцию `calculate_sum()` и разбивает общий диапазон на несколько частей.


## 2.2. Особенность вычисления суммы

Так как число `10_000_000_000_000` очень большое, считать сумму обычным циклом нецелесообразно.

Например, такой код выполнялся бы слишком долго:

```python
total = 0

for i in range(1, 10_000_000_000_000 + 1):
    total += i
```

Поэтому в работе используется формула суммы арифметической прогрессии:

```python
sum = (start + end) * (end - start + 1) // 2
```

Функция подсчёта суммы диапазона:

```python
def calculate_sum(start: int, end: int) -> int:
    return (start + end) * (end - start + 1) // 2
```


## 2.3. Разбиение задачи на части

Общий диапазон чисел от 1 до `10_000_000_000_000` делится на несколько поддиапазонов.

Например, при `WORKERS = 4` диапазон делится так:

```text
1-я часть: 1 ... 2_500_000_000_000
2-я часть: 2_500_000_000_001 ... 5_000_000_000_000
3-я часть: 5_000_000_000_001 ... 7_500_000_000_000
4-я часть: 7_500_000_000_001 ... 10_000_000_000_000
```

Каждая параллельная задача считает сумму своего диапазона. После этого частичные суммы складываются.


## 2.4. Реализация через threading

В программе `threading_task1.py` используется модуль `threading`.

Каждый поток получает свой диапазон чисел, считает сумму и записывает результат в общий список.

Основная логика:

```python
def calculate_sum(start: int, end: int) -> int:
    return (start + end) * (end - start + 1) // 2


def worker(index: int, start: int, end: int, results: list[int]) -> None:
    results[index] = calculate_sum(start, end)


def main() -> None:
    ranges = split_range(N, WORKERS)

    threads = []
    results = [0] * WORKERS

    start_time = time.perf_counter()

    for index, (start, end) in enumerate(ranges):
        thread = threading.Thread(
            target=worker,
            args=(index, start, end, results)
        )

        threads.append(thread)
        thread.start()

    for thread in threads:
        thread.join()

    total_sum = sum(results)

    end_time = time.perf_counter()
    duration = end_time - start_time

    print_result("threading", total_sum, duration)


if __name__ == "__main__":
    main()
```

Потоки удобны тем, что они используют общую память процесса. Однако для CPU-bound задачи такой подход не даёт значительного ускорения из-за GIL.

## 2.5. Реализация через multiprocessing

В программе `task1_multiprocessing_sum.py` используется модуль `multiprocessing`.

Для запуска нескольких процессов применяется `multiprocessing.Pool`.

Основная логика:

```python
def calculate_sum(start: int, end: int) -> int:
    return (start + end) * (end - start + 1) // 2


def main() -> None:
    ranges = split_range(N, WORKERS)

    start_time = time.perf_counter()

    with multiprocessing.Pool(processes=WORKERS) as pool:
        results = pool.starmap(calculate_sum, ranges)

    total_sum = sum(results)

    end_time = time.perf_counter()
    duration = end_time - start_time

    print_result("multiprocessing", total_sum, duration)


if __name__ == "__main__":
    main()
```

Каждый процесс считает сумму своего диапазона. После завершения процессов главный процесс складывает частичные результаты.

Этот подход лучше всего подходит для вычислительных задач, так как позволяет использовать несколько ядер процессора.

## 2.6. Реализация через asyncio

В программе `task1_async.py` используется `asyncio`.

Основная логика:

```python
async def calculate_sum(start: int, end: int) -> int:
    return (start + end) * (end - start + 1) // 2


async def main_async() -> None:
    ranges = split_range(N, WORKERS)

    start_time = time.perf_counter()

    tasks = [
        calculate_sum(start, end)
        for start, end in ranges
    ]

    results = await asyncio.gather(*tasks)

    total_sum = sum(results)

    end_time = time.perf_counter()
    duration = end_time - start_time

    print_result("asyncio", total_sum, duration)


if __name__ == "__main__":
    asyncio.run(main_async())
```

Асинхронный подход в данной задаче используется для сравнения. На практике `asyncio` лучше подходит не для тяжёлых вычислений, а для I/O-bound задач.


## 2.7. Результаты выполнения задачи 1

| Подход            | Время выполнения, сек |
| ----------------- | --------------------: |
| `threading`       |                `0.001027` |
| `multiprocessing` |                `0.102670` |
| `asyncio`         |                `0.000072` |


## 2.8. Вывод по задаче 1

Так как сумма считалась по математической формуле, само вычисление выполнялось очень быстро. Поэтому время выполнения в основном отражает не сложность вычислений, а накладные расходы каждого подхода.

`multiprocessing` оказался медленнее, потому что создание отдельных процессов занимает заметное время. Для такой маленькой по времени операции это дороже, чем само вычисление.

`threading` быстрее `multiprocessing` , потому что потоки легче процессов.
`asyncio` быстрее всех, потому что задачи почти сразу возвращают результат и не создаются отдельные процессы или потоки.

Для реальных тяжёлых CPU-bound задач наиболее подходящим является `multiprocessing`, так как он позволяет использовать несколько ядер процессора c отдельными ресурсами. `threading` и `asyncio` если бы использовался for уступали бы, `multiprocessing` может одновременно считать диапазоны и использует для этого больше ресурсов.


# 3. Задача 2. Параллельный парсинг веб-страниц

## 3.1. Постановка задачи

Необходимо написать три программы для параллельного парсинга нескольких веб-страниц:

* через `threading`;
* через `multiprocessing`;
* через `asyncio`.

Каждая программа должна содержать функцию:

```python
parse_and_save(url)
```

Эта функция выполняет следующие действия:

1. загружает HTML-страницу по URL;
2. извлекает заголовок страницы из тега `<title>`;
3. сохраняет результат в базу данных;
4. выводит результат на экран.


## 3.2. Использование базы данных из предыдущей лабораторной работы

В предыдущей лабораторной работе использовалась база данных PostgreSQL для FastAPI-приложения учёта личных финансов.

В этой лабораторной работе используется та же база данных:

```env
DB_ADMIN=postgresql://postgres:password@localhost:5433/finance_db
```

Так как заголовки веб-страниц не относятся напрямую к финансовым транзакциям, бюджетам или целям, была добавлена отдельная таблица:

```text
parsed_pages
```

---

## 3.3. Модель ParsedPage

В файл `models.py` была добавлена модель:

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

Назначение полей:

| Поле        | Назначение                                                     |
| ----------- | -------------------------------------------------------------- |
| `id`        | первичный ключ                                                 |
| `url`       | адрес веб-страницы                                             |
| `title`     | заголовок страницы                                             |
| `approach`  | использованный подход: `threading`, `multiprocessing`, `async` |
| `parsed_at` | дата и время парсинга                                          |


## 3.4. Миграция базы данных

После добавления модели была создана и применена миграция Alembic.

Команда создания миграции:

```bash
alembic revision --autogenerate -m "add parsed pages table"
```

Команда применения миграции:

```bash
alembic upgrade head
```

После этого в базе данных появилась таблица `parsed_pages`.

## 3.5. Асинхронное подключение к базе данных

Для обычных программ `threading` и `multiprocessing` используется синхронное подключение к базе данных.

Для async-программы было добавлено отдельное асинхронное подключение:

```env
DB_ADMIN_ASYNC=postgresql+asyncpg://postgres:password@localhost:5433/finance_db
```

В `connection.py` был добавлен `AsyncSessionLocal`:

```python
async_engine = create_async_engine(db_async_url, echo=True)

AsyncSessionLocal = async_sessionmaker(
    async_engine,
    class_=AsyncSession,
    expire_on_commit=False
)
```

Это нужно для того, чтобы в async-версии не блокировать event loop обычными синхронными запросами к базе данных.

## 3.6. Список URL-адресов

Для парсинга использовался список сайтов:

```python
URLS = [
    "https://example.com",
    "https://www.python.org",
    "https://docs.python.org/3/",
    "https://fastapi.tiangolo.com/",
    "https://www.postgresql.org/",
    "https://www.sqlalchemy.org/",
]
```


## 3.7. Общие функции парсинга

В файл `parser.py` были вынесены общие функции.

Функция получения заголовка страницы:

```python
def extract_title(html: str) -> str:
    soup = BeautifulSoup(html, "html.parser")

    if soup.title and soup.title.string:
        return soup.title.string.strip()[:500]

    return "Без заголовка"
```

Функция синхронного сохранения:

```python
def save_page_sync(url: str, title: str, approach: str) -> None:
    with Session(engine) as session:
        page = ParsedPage(
            url=url[:500],
            title=title[:500],
            approach=approach
        )

        session.add(page)
        session.commit()
```

Функция асинхронного сохранения:

```python
async def save_page_async(url: str, title: str, approach: str) -> None:
    async with AsyncSessionLocal() as session:
        page = ParsedPage(
            url=url[:500],
            title=title[:500],
            approach=approach
        )

        session.add(page)
        await session.commit()
```

## 3.8. Реализация через threading

В программе `task2_threading_parser.py`

Список URL делится на несколько частей. Для каждой части создаётся отдельный поток.

Каждый поток последовательно обрабатывает свою часть URL:

```python
APPROACH = "threading"


def parse_and_save(url: str) -> None:
    try:
        response = requests.get(url, headers=HEADERS, timeout=10)
        response.raise_for_status()

        title = extract_title(response.text)
        save_page_sync(url, title, APPROACH)


    except Exception as error:
        print(f"[{APPROACH}] Ошибка при обработке {url}: {error}")


def worker(urls: list[str]) -> None:
    """
    Функция для одного потока.
    Поток получает свою часть URL и обрабатывает их по очереди.
    """
    for url in urls:
        parse_and_save(url)


def main() -> None:
    url_parts = split_list(URLS, WORKERS)
    threads = []

    start_time = time.perf_counter()

    for part in url_parts:
        thread = threading.Thread(target=worker, args=(part,))
        threads.append(thread)
        thread.start()

    for thread in threads:
        thread.join()

    end_time = time.perf_counter()
    duration = end_time - start_time

    print(f"Время выполнения threading: {duration:.4f} секунд")


if __name__ == "__main__":
    main()

```

Функция `parse_and_save()` выполняет HTTP-запрос, извлекает заголовок страницы и сохраняет данные в PostgreSQL.

Этот подход хорошо подходит для сетевых задач, так как пока один поток ждёт ответ сайта, другой поток может выполнять свой запрос.


## 3.9. Реализация через multiprocessing

В программе `task2_multiprocessing_parser.py`

Список URL также делится на части, но обработка выполняется не потоками, а отдельными процессами.

Основная логика:

```python
APPROACH = "multiprocessing"


def parse_and_save(url: str) -> None:
    try:
        response = requests.get(url, headers=HEADERS, timeout=10)
        response.raise_for_status()

        title = extract_title(response.text)
        save_page_sync(url, title, APPROACH)

    except Exception as error:
        print(f"[{APPROACH}] Ошибка при обработке {url}: {error}")


def worker(urls: list[str]) -> None:
    """
    Функция для одного процесса.
    Процесс получает свою часть URL и обрабатывает их по очереди.
    """
    for url in urls:
        parse_and_save(url)


def main() -> None:
    url_parts = split_list(URLS, WORKERS)

    start_time = time.perf_counter()

    with multiprocessing.Pool(processes=WORKERS) as pool:
        pool.map(worker, url_parts)

    end_time = time.perf_counter()
    duration = end_time - start_time

    print(f"Время выполнения multiprocessing: {duration:.4f} секунд")


if __name__ == "__main__":
    main()
```

Каждый процесс обрабатывает свою часть URL и самостоятельно сохраняет данные в базу данных.

Этот подход работает, но для сетевого парсинга может иметь лишние накладные расходы, так как создание процессов тяжелее, чем создание потоков или асинхронных задач.



## 3.10. Реализация через asyncio

В программе `task2_async_parser.py`

Основная логика:

```python
APPROACH = "async"


async def parse_and_save(url: str, session: aiohttp.ClientSession) -> None:
    """
    Асинхронно загружает страницу, достаёт title и сохраняет результат в БД.
    """
    try:
        async with session.get(url, timeout=10) as response:
            response.raise_for_status()
            html = await response.text()

        title = extract_title(html)
        await save_page_async(url, title, APPROACH)


    except Exception as error:
        print(f"[{APPROACH}] Ошибка при обработке {url}: {error}")


async def main_async() -> None:
    start_time = time.perf_counter()

    async with aiohttp.ClientSession(headers=HEADERS) as session:
        tasks = [
            parse_and_save(url, session)
            for url in URLS
        ]

        await asyncio.gather(*tasks)

    end_time = time.perf_counter()
    duration = end_time - start_time

    print(f"Время выполнения async: {duration:.4f} секунд")


if __name__ == "__main__":
    asyncio.run(main_async())
```

Для каждого URL создаётся асинхронная задача:

```python
tasks = [
    parse_and_save(url, session)
    for url in URLS
]
```

Затем все задачи запускаются конкурентно:

```python
await asyncio.gather(*tasks)
```

HTTP-запросы выполняются через `aiohttp.ClientSession`, а запись в базу данных выполняется через асинхронную сессию SQLAlchemy.

Этот подход хорошо подходит для большого количества сетевых запросов, так как event loop переключается между задачами в моменты ожидания ответа сайта или базы данных.

## 3.11. Результаты выполнения задачи 2

| Подход            | Количество URL | Время выполнения, сек |
| ----------------- | -------------: | --------------------: |
| `threading`       |              6 |                `0.9879` |
| `multiprocessing` |              6 |                `1.4594` |
| `asyncio`         |              6 |                `0.6402` |

Парсинг сайтов это I/O-bound задача. Программа много времени ждёт ответ сайта, загрузку html, запись в бд. Поэтому `asyncio` показал лучший результат. Пока одна асинхронная задача ждёт ответ сайта, event loop переключается на другую задачу. 

`threading` тоже показал хороший результат, потому что потоки подходят для сетевых запросов. Пока один поток ждёт ответ сайта, другой может работать.

`multiprocessing` оказался самым медленным, потому что для парсинга сайтов создание отдельных процессов часто даёт лишние накладные расходы. Процессы полезнее для тяжёлых вычислений, а не для сетевого ожидания.


# 4. Сравнение подходов

## 4.1. Сравнение по первой задаче

| Подход            | Особенности                                           | Подходит для вычислений |
| ----------------- | ----------------------------------------------------- | ----------------------- |
| `threading`       | потоки внутри одного процесса, есть ограничение GIL   | Не лучший вариант       |
| `multiprocessing` | отдельные процессы, можно использовать несколько ядер | Да                      |
| `asyncio`         | конкурентное выполнение через event loop              | Не лучший вариант       |

Для вычислительных задач наиболее подходящим является `multiprocessing`.



## 4.2. Сравнение по второй задаче

| Подход            | Особенности                                          | Подходит для парсинга |
| ----------------- | ---------------------------------------------------- | --------------------- |
| `threading`       | удобно использовать для сетевых запросов             | Да                    |
| `multiprocessing` | работает, но имеет большие накладные расходы         | Можно использовать    |
| `asyncio`         | хорошо подходит для большого количества I/O-операций | Да                    |

Для парсинга веб-страниц лучше всего подходят `threading` и `asyncio`, так как задача является I/O-bound.


# 5. Общий вывод

В ходе лабораторной работы были реализованы программы с использованием трёх подходов к параллельному и конкурентному выполнению задач в Python: `threading`, `multiprocessing` и `asyncio`.

По результатам работы можно сделать вывод, что выбор подхода зависит от типа задачи. Для вычислительных CPU-bound задач лучше подходит `multiprocessing`, так как он позволяет использовать несколько процессов и несколько ядер процессора. Для I/O-bound задач, таких как парсинг сайтов и работа с сетью, хорошо подходят `threading` и `asyncio`. При этом `asyncio` особенно удобно использовать вместе с асинхронными библиотеками, например `aiohttp` и асинхронным подключением к базе данных.


