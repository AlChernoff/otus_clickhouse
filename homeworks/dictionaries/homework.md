# Домашнее задание по ClickHouse: таблица, словарь и оконная функция

## Задание

Создать таблицу с полями:

```sql
user_id UInt64,
action String,
expense UInt64
```

Создать словарь, где:

- ключ — `user_id`;
- атрибут — `email String`;
- источник словаря — файл.

Наполнить таблицу и источник данными с низкоардинальными значениями для поля `action` и несколькими повторяющимися строками для каждого `user_id`.

Написать `SELECT`, который возвращает:

- `email` с помощью `dictGet`;
- аккумулятивную сумму `expense` с окном по `action`;
- сортировку по `email`.

---

## 1. Создание таблицы

```sql
CREATE TABLE user_actions
(
    user_id UInt64,
    action String,
    expense UInt64
)
ENGINE = MergeTree
ORDER BY (action, user_id);
```

---

## 2. Создание файла для словаря

Файл был создан по пути:

```text
/var/lib/clickhouse/user_files/users.csv
```

Содержимое файла:

```csv
1,alice@example.com
2,bob@example.com
3,charlie@example.com
```

---

## 3. Создание словаря

```sql
CREATE DICTIONARY user_email_dict
(
    user_id UInt64,
    email String
)
PRIMARY KEY user_id
SOURCE(FILE(
    path '/var/lib/clickhouse/user_files/users.csv'
    format 'CSV'
))
LAYOUT(FLAT())
LIFETIME(300);
```

ClickHouse успешно создал словарь:

```text
Ok.

0 rows in set. Elapsed: 0.009 sec.
```

---

## 4. Проверка работы словаря

```sql
SELECT dictGet('user_email_dict', 'email', 1);
```

Результат:

```text
┌─dictGet('user_email_dict', 'email', 1)─┐
│ alice@example.com                      │
└────────────────────────────────────────┘
```

Это означает, что словарь корректно возвращает `email` по ключу `user_id`.

---

## 5. Наполнение таблицы данными

```sql
INSERT INTO user_actions VALUES
(1, 'click',    10),
(1, 'click',    20),
(1, 'view',      5),
(1, 'purchase', 80),

(2, 'click',    15),
(2, 'click',    25),
(2, 'view',      7),
(2, 'purchase', 100),

(3, 'click',    12),
(3, 'view',      8),
(3, 'view',      9),
(3, 'purchase', 120);
```

Результат выполнения:

```text
Ok.

12 rows in set. Elapsed: 0.004 sec.
```

В поле `action` используются низкоардинальные значения:

```text
click
view
purchase
```

Для каждого `user_id` есть несколько повторяющихся строк.

---

## 6. Итоговый SELECT-запрос

```sql
SELECT
    dictGet('user_email_dict', 'email', user_id) AS email,
    user_id,
    action,
    expense,
    sum(expense) OVER (
        PARTITION BY action
        ORDER BY user_id
    ) AS cumulative_expense_by_action
FROM user_actions
ORDER BY email;
```

---

## 7. Результат выполнения SELECT

```text
┌─email───────────────┬─user_id─┬─action───┬─expense─┬─cumulative_expense_by_action─┐
│ alice@example.com   │       1 │ click    │      10 │                           30 │
│ alice@example.com   │       1 │ click    │      20 │                           30 │
│ alice@example.com   │       1 │ purchase │      80 │                           80 │
│ alice@example.com   │       1 │ view     │       5 │                            5 │
│ bob@example.com     │       2 │ click    │      15 │                           70 │
│ bob@example.com     │       2 │ click    │      25 │                           70 │
│ bob@example.com     │       2 │ purchase │     100 │                          180 │
│ bob@example.com     │       2 │ view     │       7 │                           12 │
│ charlie@example.com │       3 │ click    │      12 │                           82 │
│ charlie@example.com │       3 │ purchase │     120 │                          300 │
│ charlie@example.com │       3 │ view     │       8 │                           29 │
│ charlie@example.com │       3 │ view     │       9 │                           29 │
└─────────────────────┴─────────┴──────────┴─────────┴──────────────────────────────┘
```

---

## 8. Пояснение запроса

В запросе используется функция:

```sql
dictGet('user_email_dict', 'email', user_id)
```

Она получает значение `email` из словаря `user_email_dict` по ключу `user_id`.

Оконная функция:

```sql
sum(expense) OVER (
    PARTITION BY action
    ORDER BY user_id
)
```

считает накопительную сумму `expense` отдельно для каждой группы `action`.

Часть:

```sql
PARTITION BY action
```

разбивает данные на отдельные группы:

- `click`;
- `view`;
- `purchase`.

Внутри каждой такой группы сумма считается отдельно.

Часть:

```sql
ORDER BY user_id
```

задает порядок строк внутри каждой партиции.

Финальная часть:

```sql
ORDER BY email
```

сортирует итоговый результат по адресу электронной почты.

---

## Итог

В результате были выполнены все пункты задания:

- создана таблица `user_actions`;
- создан файловый словарь `user_email_dict`;
- словарь проверен с помощью `dictGet`;
- таблица наполнена тестовыми данными;
- написан `SELECT`, который возвращает `email`, данные из таблицы и аккумулятивную сумму `expense` по `action`;
- результат отсортирован по `email`.
