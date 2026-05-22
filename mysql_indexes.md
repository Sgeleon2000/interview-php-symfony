# Индексы в MySQL — полный справочник

**Индекс** — отдельная структура данных, которая ускоряет поиск строк в таблице ценой замедления INSERT/UPDATE/DELETE и дополнительного места на диске.

Без индекса MySQL делает **full table scan** — читает все строки. С индексом — идёт по дереву поиска и читает только нужные.

---

## Классификация индексов

Индексы в MySQL можно классифицировать по нескольким осям:

| Ось                     | Виды                                                       |
| ----------------------- | ---------------------------------------------------------- |
| **Структура данных**    | B-Tree, Hash, R-Tree (Spatial), Full-Text, Inverted        |
| **Назначение / роль**   | Primary, Unique, Regular, Foreign Key                      |
| **Состав колонок**      | Single-column, Composite (Multi-column), Prefix, Functional |
| **Хранение строк**      | Clustered, Secondary (Non-clustered)                       |
| **Видимость**           | Visible, Invisible                                          |
| **Покрытие**            | Covering, Non-covering                                     |

---

## 1. По структуре данных

### 1.1. B-Tree (B+Tree) — дефолтный

Сбалансированное дерево. Используется по умолчанию в InnoDB и MyISAM для большинства типов индексов.

```sql
-- Просто индекс — по умолчанию это B-Tree
CREATE INDEX idx_email ON users (email);

-- Явное указание (не обязательно)
CREATE INDEX idx_email ON users (email) USING BTREE;
```

**Когда работает:**
- Равенство: `WHERE email = 'a@b.com'`
- Диапазоны: `WHERE age > 18`, `BETWEEN`, `<`, `>`, `<=`, `>=`
- Префиксный поиск по строкам: `WHERE name LIKE 'Иван%'` (но НЕ `LIKE '%Иван'`)
- Сортировка `ORDER BY` и группировка `GROUP BY`

**Когда НЕ работает:**
- `LIKE '%text%'` (поиск с подстановкой в начале)
- Применение функций к колонке: `WHERE YEAR(created_at) = 2025` (нужен functional index)
- `NOT IN`, `<>` в большинстве случаев

### 1.2. Hash

Хэш-таблица. Поддерживается только в движке **MEMORY** (и InnoDB Adaptive Hash Index — автоматически).

```sql
CREATE TABLE cache (
    key_hash CHAR(32),
    value TEXT,
    INDEX idx_key (key_hash) USING HASH
) ENGINE = MEMORY;
```

**Плюсы:** очень быстрый поиск по точному равенству — O(1).
**Минусы:**
- Только `=` и `IN`. Диапазоны не работают.
- Нет сортировки.
- Хэш-коллизии могут замедлять.

В InnoDB сам сервер автоматически строит **Adaptive Hash Index** в памяти для горячих B-Tree-страниц — это включено по умолчанию.

### 1.3. R-Tree (Spatial Index)

Для геоданных — точек, полигонов, регионов. Работает с типами `GEOMETRY`, `POINT`, `POLYGON`.

```sql
CREATE TABLE places (
    id INT PRIMARY KEY,
    location POINT NOT NULL,
    SPATIAL INDEX idx_location (location)
) ENGINE = InnoDB;

-- Запрос: найти точки в радиусе
SELECT * FROM places
WHERE ST_Distance_Sphere(location, POINT(37.6, 55.7)) < 1000;
```

Используется для пространственных функций: `ST_Contains()`, `ST_Within()`, `ST_Distance()` и т.д.

### 1.4. Full-Text Index

Для полнотекстового поиска по `CHAR`/`VARCHAR`/`TEXT`. Не B-Tree, а **инвертированный индекс**.

```sql
CREATE TABLE articles (
    id INT PRIMARY KEY,
    title VARCHAR(255),
    body TEXT,
    FULLTEXT INDEX idx_search (title, body)
) ENGINE = InnoDB;

-- Поиск в natural language mode (по умолчанию)
SELECT * FROM articles
WHERE MATCH(title, body) AGAINST('mysql index');

-- Boolean mode — поддерживает операторы +, -, *, "..."
SELECT * FROM articles
WHERE MATCH(title, body) AGAINST('+mysql -postgresql' IN BOOLEAN MODE);
```

**Особенности:**
- Минимальная длина слова: `innodb_ft_min_token_size` (по умолчанию 3)
- Стоп-слова игнорируются
- Поддерживает русский / китайский с помощью ngram-парсера
- Используется ТОЛЬКО через `MATCH() AGAINST()`

### 1.5. Сводная таблица по структурам

| Структура  | Где                  | Что умеет                          | Что не умеет                |
| ---------- | -------------------- | ---------------------------------- | --------------------------- |
| B-Tree     | InnoDB, MyISAM       | =, >, <, BETWEEN, LIKE 'x%', ORDER BY | LIKE '%x', функции от поля |
| Hash       | MEMORY (явно)        | =, IN                              | диапазоны, ORDER BY         |
| R-Tree     | InnoDB, MyISAM       | пространственные функции           | обычные сравнения           |
| Full-Text  | InnoDB, MyISAM       | MATCH ... AGAINST                  | обычные WHERE-условия       |

---

## 2. По назначению

### 2.1. PRIMARY KEY — первичный ключ

Главный индекс таблицы. Уникальный, не-NULL, **только один на таблицу**.

В InnoDB он же — **clustered index** (строки физически отсортированы по нему).

```sql
-- При создании таблицы
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) NOT NULL
);

-- Композитный PK (составной)
CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);

-- Добавить позже
ALTER TABLE users ADD PRIMARY KEY (id);
```

### 2.2. UNIQUE INDEX — уникальный

Гарантирует уникальность значений. Может содержать NULL (несколько NULL допускаются).

```sql
CREATE UNIQUE INDEX uniq_email ON users (email);

-- Или при создании таблицы
CREATE TABLE users (
    id INT PRIMARY KEY,
    email VARCHAR(255),
    UNIQUE KEY uniq_email (email)
);

-- Композитный UNIQUE: пара (user_id, day) уникальна
CREATE UNIQUE INDEX uniq_attendance ON attendance (user_id, day);
```

### 2.3. INDEX / KEY — обычный (non-unique)

Просто ускоряет поиск, без ограничений уникальности.

```sql
CREATE INDEX idx_name ON users (name);

-- Эквивалентно:
ALTER TABLE users ADD INDEX idx_name (name);
ALTER TABLE users ADD KEY idx_name (name);  -- KEY = INDEX
```

### 2.4. FOREIGN KEY — внешний ключ

Не совсем индекс, но MySQL автоматически создаёт обычный индекс под FK, если его нет.

```sql
CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT,
    FOREIGN KEY (user_id) REFERENCES users(id)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);
```

---

## 3. По составу колонок

### 3.1. Single-column — по одной колонке

```sql
CREATE INDEX idx_email ON users (email);
```

### 3.2. Composite (Multi-column) — составной

По нескольким колонкам. **Порядок важен!**

```sql
CREATE INDEX idx_name_age ON users (last_name, first_name, age);
```

**Правило "leftmost prefix":** индекс работает, если в WHERE используется **префикс** колонок слева направо.

```sql
-- ✓ Работает (используется last_name)
WHERE last_name = 'Иванов'

-- ✓ Работает (last_name + first_name)
WHERE last_name = 'Иванов' AND first_name = 'Иван'

-- ✓ Работает полностью
WHERE last_name = 'Иванов' AND first_name = 'Иван' AND age = 30

-- ✗ НЕ работает (нет last_name в начале)
WHERE first_name = 'Иван'

-- ✗ НЕ работает (пропущен first_name между)
WHERE last_name = 'Иванов' AND age = 30  -- last_name работает, age — нет
```

**Как выбирать порядок колонок:**
1. Колонки с равенством (`=`) — в начало
2. Колонки с высокой селективностью (много разных значений) — раньше
3. Колонки в `ORDER BY` — в конец (если они в том же порядке)

### 3.3. Prefix Index — индекс по префиксу строки

Полезно для длинных VARCHAR/TEXT — индексируем только первые N символов.

```sql
-- Индекс по первым 10 символам email
CREATE INDEX idx_email_prefix ON users (email(10));

-- Для TEXT/BLOB длина обязательна
CREATE INDEX idx_body_prefix ON articles (body(50));
```

**Плюсы:** маленький индекс, быстрее обновления, меньше места.
**Минусы:** не может использоваться для покрывающего поиска, нельзя сортировать.

Как выбрать длину префикса:
```sql
-- Посмотреть селективность префиксов
SELECT
    COUNT(DISTINCT LEFT(email, 5)) / COUNT(*) AS sel5,
    COUNT(DISTINCT LEFT(email, 10)) / COUNT(*) AS sel10,
    COUNT(DISTINCT LEFT(email, 15)) / COUNT(*) AS sel15
FROM users;
-- Чем ближе к 1.0 — тем лучше префикс отделяет записи
```

### 3.4. Functional Index (MySQL 8.0+) — по выражению

Индекс по результату функции/выражения.

```sql
-- До MySQL 8.0 такой запрос НЕ использовал индекс по email:
WHERE LOWER(email) = 'user@example.com'

-- С MySQL 8.0 можно сделать функциональный индекс:
CREATE INDEX idx_email_lower ON users ((LOWER(email)));
-- Заметь двойные скобки — обязательны!

-- Примеры
CREATE INDEX idx_year ON orders ((YEAR(created_at)));
CREATE INDEX idx_jpath ON products ((CAST(JSON_EXTRACT(data, '$.sku') AS CHAR(50))));
```

### 3.5. Descending Index (MySQL 8.0+) — по убыванию

Раньше `DESC` в индексах игнорировался. С 8.0 — учитывается.

```sql
-- Полезно, когда часто сортируешь по убыванию ИЛИ комбинируешь ASC/DESC
CREATE INDEX idx_score_date ON scores (score DESC, created_at ASC);

-- Запрос:
SELECT * FROM scores ORDER BY score DESC, created_at ASC;
-- Использует индекс без filesort
```

---

## 4. По хранению строк (только InnoDB)

### 4.1. Clustered Index — кластеризованный

**Сами строки** таблицы упорядочены по этому индексу — данные и индекс это одна структура.

В InnoDB это **всегда первичный ключ** (или первый UNIQUE NOT NULL, или внутренний `GEN_CLUST_INDEX`).

```text
Clustered (по PRIMARY KEY id):
   id=1  → (id=1, email='a@b.com', name='Alice')
   id=2  → (id=2, email='b@c.com', name='Bob')
   id=3  → (id=3, email='c@d.com', name='Carol')

Поиск по PK = найти лист дерева → строка уже там, дополнительный lookup не нужен.
```

**Следствия:**
- Поиск по PRIMARY KEY — самый быстрый
- INSERT в "случайные" места (UUID без сортировки) — медленные, фрагментация
- Лучший PK — короткий, постоянно растущий (`AUTO_INCREMENT`, UUIDv7)

### 4.2. Secondary Index — вторичный

Любой другой индекс. Хранит **значение индексируемых колонок + PRIMARY KEY** (а не указатель на строку).

```text
Secondary index по email:
   'a@b.com' → PK=1
   'b@c.com' → PK=2
   'c@d.com' → PK=3

Поиск по email = найти PK в secondary → потом ещё один поиск в clustered → строка.
Это называется "double lookup" или "bookmark lookup".
```

**Следствие:** длинный PRIMARY KEY раздувает ВСЕ secondary-индексы.

---

## 5. По видимости (MySQL 8.0+)

### 5.1. Visible (по умолчанию)

Оптимизатор учитывает индекс.

### 5.2. Invisible — невидимый

Индекс существует, поддерживается при INSERT/UPDATE/DELETE, но **оптимизатор его игнорирует**. Очень удобно перед удалением индекса проверить, ничего ли не сломается.

```sql
-- Сделать индекс невидимым (мониторим перфоманс)
ALTER TABLE users ALTER INDEX idx_old_field INVISIBLE;

-- Если всё ок — дропаем
ALTER TABLE users DROP INDEX idx_old_field;

-- Если что-то сломалось — мгновенно возвращаем
ALTER TABLE users ALTER INDEX idx_old_field VISIBLE;
```

Можно проверить через `EXPLAIN`, что индекс не используется.

---

## 6. Covering Index — покрывающий

Не отдельный тип, а **свойство**: индекс содержит ВСЕ колонки, нужные для запроса, поэтому идти за строкой в clustered index не нужно.

```sql
-- Индекс
CREATE INDEX idx_user_status ON orders (user_id, status, created_at);

-- Этот запрос — покрывающий (в EXPLAIN будет Extra: "Using index")
SELECT user_id, status, created_at
FROM orders
WHERE user_id = 42;

-- А этот — нет (нужна ещё колонка total, придётся идти в строку)
SELECT user_id, status, total
FROM orders
WHERE user_id = 42;
```

**Зачем:** Covering index убирает второй lookup в кластерный индекс — это часто двукратное ускорение.

---

## 7. Управление индексами

### Создание

```sql
-- В составе CREATE TABLE
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) NOT NULL,
    name VARCHAR(100),

    UNIQUE KEY uniq_email (email),
    INDEX idx_name (name),
    INDEX idx_name_email (name, email)
);

-- Или отдельно
CREATE INDEX idx_name ON users (name);
CREATE UNIQUE INDEX uniq_email ON users (email);
CREATE FULLTEXT INDEX idx_search ON articles (title, body);
CREATE SPATIAL INDEX idx_loc ON places (location);

-- Через ALTER TABLE
ALTER TABLE users
    ADD INDEX idx_name (name),
    ADD UNIQUE KEY uniq_email (email);
```

### Удаление

```sql
DROP INDEX idx_name ON users;
ALTER TABLE users DROP INDEX idx_name;
ALTER TABLE users DROP PRIMARY KEY;
```

### Просмотр

```sql
-- Все индексы таблицы
SHOW INDEX FROM users;

-- Через information_schema
SELECT * FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'users';

-- Размер индексов
SELECT
    TABLE_NAME,
    INDEX_NAME,
    ROUND(STAT_VALUE * @@innodb_page_size / 1024 / 1024, 2) AS size_mb
FROM mysql.innodb_index_stats
WHERE database_name = 'mydb' AND stat_name = 'size'
ORDER BY size_mb DESC;
```

### Анализ использования

```sql
-- Какие индексы используются для запроса
EXPLAIN SELECT * FROM users WHERE email = 'a@b.com';
-- Смотри столбцы: key, key_len, rows, Extra ('Using index' = covering)

-- В MySQL 8 — детальнее
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'a@b.com';

-- Какие индексы НЕ использовались (полезно для очистки)
SELECT
    object_schema, object_name, index_name
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE index_name IS NOT NULL
    AND count_star = 0
    AND object_schema NOT IN ('mysql', 'performance_schema', 'sys');
```

---

## 8. Сводная таблица всех индексов

| Тип                       | Структура | Уникальность | Где используется                       |
| ------------------------- | --------- | ------------ | -------------------------------------- |
| **PRIMARY KEY**           | B-Tree    | Да           | Один на таблицу, обязательный (de facto) |
| **UNIQUE INDEX**          | B-Tree    | Да           | Когда значения должны быть уникальны   |
| **INDEX / KEY**           | B-Tree    | Нет          | Обычное ускорение поиска               |
| **FOREIGN KEY**           | B-Tree    | Нет          | Связи между таблицами                  |
| **Composite**             | B-Tree    | Любая        | Запросы по нескольким колонкам         |
| **Prefix Index**          | B-Tree    | Любая        | Длинные строки/тексты                  |
| **Functional Index** (8.0+) | B-Tree  | Любая        | Условия с функциями: `LOWER()`, `YEAR()` |
| **Descending Index** (8.0+) | B-Tree  | Любая        | `ORDER BY ... DESC`                    |
| **Hash Index**            | Hash      | Любая        | Движок MEMORY, только `=`              |
| **Spatial Index**         | R-Tree    | Нет          | Геоданные                              |
| **Full-Text Index**       | Inverted  | Нет          | Текстовый поиск через `MATCH AGAINST`  |
| **Clustered Index**       | B-Tree    | Да           | InnoDB: всегда PRIMARY KEY             |
| **Secondary Index**       | B-Tree    | Любая        | Любой не-clustered индекс              |
| **Covering Index**        | B-Tree    | Любая        | Свойство: индекс содержит всё для SELECT |
| **Invisible Index** (8.0+) | Любая    | Любая        | Скрыт от оптимизатора                  |

---

## 9. Когда индекс НЕ помогает (и даже мешает)

```text
✗ Запрос возвращает БОЛЬШУЮ часть таблицы
  WHERE active = 1  -- если 90% строк active=1, scan дешевле

✗ Селективность колонки очень низкая
  Индекс по полу (M/F/Other) обычно бесполезен

✗ Функция оборачивает колонку (до 8.0)
  WHERE YEAR(created_at) = 2025  -- нужен functional index

✗ Колонка слева в LIKE
  WHERE name LIKE '%иван%'

✗ Неявные приведения типов
  WHERE phone = 79991234567  -- если phone VARCHAR, индекс может не сработать

✗ OR между разными колонками без индекса
  WHERE (a = 1 OR b = 2)  -- часто не использует индексы. Лучше UNION

✗ Маленькая таблица — full scan быстрее похода в индекс

ПЛЮС, каждый индекс ЗАМЕДЛЯЕТ:
  • INSERT (нужно обновить все индексы)
  • UPDATE индексируемых колонок
  • DELETE
  • Занимает место на диске и в RAM (buffer pool)
```

---

## 10. Best Practices

```text
✓ PRIMARY KEY — короткий, постоянно растущий (BIGINT AUTO_INCREMENT, UUIDv7).
✓ Покрывай частые WHERE/JOIN/ORDER BY индексами.
✓ В композитных индексах: сначала равенство, потом диапазон, потом ORDER BY.
✓ Используй EXPLAIN, чтобы проверять, что индекс реально используется.
✓ Перед удалением индекса делай его INVISIBLE и смотри метрики.
✓ Покрывающие индексы (covering) — частый способ ускорить SELECT в 2 раза.
✓ Для длинных строк — prefix index.
✓ Для геоданных — SPATIAL.
✓ Для текста — FULLTEXT.

✗ Не индексируй "на всякий случай" — каждый индекс замедляет запись.
✗ Не индексируй маленькие колонки с низкой селективностью (gender, is_active).
✗ Не делай PRIMARY KEY по случайному UUID v4 — это убивает clustered index.
✗ Не оборачивай колонки в функции без functional index (8.0+).
✗ Не дублируй индексы: idx(a) и idx(a,b) — первый обычно не нужен.
✗ Не забывай про статистику: ANALYZE TABLE при больших изменениях данных.
```

---

## 11. Полезные команды

```sql
-- Перестроить статистику (оптимизатор использует её)
ANALYZE TABLE users;

-- Оптимизировать таблицу (дефрагментация, перестроение индексов)
OPTIMIZE TABLE users;

-- Найти дубликаты индексов (не работает в стандартном MySQL — нужен pt-duplicate-key-checker из Percona Toolkit)
pt-duplicate-key-checker --host=localhost --user=root mydb

-- Размер всех индексов по таблицам
SELECT
    TABLE_NAME,
    ROUND(DATA_LENGTH / 1024 / 1024, 2) AS data_mb,
    ROUND(INDEX_LENGTH / 1024 / 1024, 2) AS index_mb,
    ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS total_mb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'mydb'
ORDER BY total_mb DESC;
```