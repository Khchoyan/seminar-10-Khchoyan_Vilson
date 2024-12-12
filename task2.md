# Задание 2: Специальные случаи использования индексов

# Партиционирование и специальные случаи использования индексов

1. Удалите прошлый инстанс PostgreSQL - `docker-compose down` в папке `src` и запустите новый: `docker-compose up -d`.

2. Создайте партиционированную таблицу и заполните её данными:

    ```sql
    -- Создание партиционированной таблицы
    CREATE TABLE t_books_part (
        book_id     INTEGER      NOT NULL,
        title       VARCHAR(100) NOT NULL,
        category    VARCHAR(30),
        author      VARCHAR(100) NOT NULL,
        is_active   BOOLEAN      NOT NULL
    ) PARTITION BY RANGE (book_id);

    -- Создание партиций
    CREATE TABLE t_books_part_1 PARTITION OF t_books_part
        FOR VALUES FROM (MINVALUE) TO (50000);

    CREATE TABLE t_books_part_2 PARTITION OF t_books_part
        FOR VALUES FROM (50000) TO (100000);

    CREATE TABLE t_books_part_3 PARTITION OF t_books_part
        FOR VALUES FROM (100000) TO (MAXVALUE);

    -- Копирование данных из t_books
    INSERT INTO t_books_part 
    SELECT * FROM t_books;
    ```

3. Обновите статистику таблиц:
   ```sql
   ANALYZE t_books;
   ANALYZE t_books_part;
   ```
   
   *Результат:*
    Query returned successfully in 2 secs 393 msec.
   
5. Выполните запрос для поиска книги с id = 18:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part WHERE book_id = 18;
   ```
   
   *План выполнения:*
  "Seq Scan on t_books_part_1 t_books_part  (cost=0.00..1032.99 rows=1 width=32) (actual time=0.013..6.190 rows=1 loops=1)"
  "  Filter: (book_id = 18)"
  "  Rows Removed by Filter: 49998"
  "Planning Time: 0.646 ms"
  "Execution Time: 6.226 ms"
   
   *Объясните результат:*
   Индекс не используется, так как происходит поиск по Id

6. Выполните поиск по названию книги:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part 
   WHERE title = 'Expert PostgreSQL Architecture';
   ```
   
   *План выполнения:*
   "Append  (cost=0.00..3101.01 rows=3 width=33) (actual time=9.308..26.582 rows=1 loops=1)"
    "  ->  Seq Scan on t_books_part_1  (cost=0.00..1032.99 rows=1 width=32) (actual time=9.307..9.333 rows=1 loops=1)"
    "        Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
    "        Rows Removed by Filter: 49998"
    "  ->  Seq Scan on t_books_part_2  (cost=0.00..1034.00 rows=1 width=33) (actual time=7.612..7.613 rows=0 loops=1)"
    "        Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
    "        Rows Removed by Filter: 50000"
    "  ->  Seq Scan on t_books_part_3  (cost=0.00..1034.01 rows=1 width=34) (actual time=9.628..9.628 rows=0 loops=1)"
    "        Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
    "        Rows Removed by Filter: 50001"
    "Planning Time: 0.865 ms"
    "Execution Time: 26.632 ms"
   
   *Объясните результат:*
   В новой таблице нет индексов следовательно происходит последовательное сканирование

7. Создайте партиционированный индекс:
   ```sql
   CREATE INDEX ON t_books_part(title);
   ```
   
   *Результат:*
    Query returned successfully in 2 secs 706 msec.
   
9. Повторите запрос из шага 5:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part 
   WHERE title = 'Expert PostgreSQL Architecture';
   ```
   
   *План выполнения:*
   "Append  (cost=0.29..24.94 rows=3 width=33) (actual time=0.059..0.119 rows=1 loops=1)"
    "  ->  Index Scan using t_books_part_1_title_idx on t_books_part_1  (cost=0.29..8.31 rows=1 width=32) (actual time=0.058..0.059 rows=1 loops=1)"
    "        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
    "  ->  Index Scan using t_books_part_2_title_idx on t_books_part_2  (cost=0.29..8.31 rows=1 width=33) (actual time=0.029..0.029 rows=0 loops=1)"
    "        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
    "  ->  Index Scan using t_books_part_3_title_idx on t_books_part_3  (cost=0.29..8.31 rows=1 width=34) (actual time=0.028..0.028 rows=0 loops=1)"
    "        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
    "Planning Time: 0.990 ms"
    "Execution Time: 0.159 ms"
   
   *Объясните результат:*
   Индекс ускорит запрос. Для поиска по каждой партиции будет использоваться индекс, потому что индекс оптимизировал запросы по заголовку

10. Удалите созданный индекс:
   ```sql
   DROP INDEX t_books_part_title_idx;
   ```
   
   *Результат:*
    Query returned successfully in 117 msec.
    
11. Создайте индекс для каждой партиции:
   ```sql
   CREATE INDEX ON t_books_part_1(title);
   CREATE INDEX ON t_books_part_2(title);
   CREATE INDEX ON t_books_part_3(title);
   ```
   
   *Результат:*
    Query returned successfully in 705 msec.

11. Повторите запрос из шага 5:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books_part 
    WHERE title = 'Expert PostgreSQL Architecture';
    ```
    
    *План выполнения:*
    "Append  (cost=0.29..24.94 rows=3 width=33) (actual time=0.102..0.175 rows=1 loops=1)"
    "  ->  Index Scan using t_books_part_1_title_idx on t_books_part_1  (cost=0.29..8.31 rows=1 width=32) (actual time=0.100..0.103 rows=1 loops=1)"
    "        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
    "  ->  Index Scan using t_books_part_2_title_idx on t_books_part_2  (cost=0.29..8.31 rows=1 width=33) (actual time=0.038..0.038 rows=0 loops=1)"
    "        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
    "  ->  Index Scan using t_books_part_3_title_idx on t_books_part_3  (cost=0.29..8.31 rows=1 width=34) (actual time=0.030..0.030 rows=0 loops=1)"
    "        Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
    "Planning Time: 3.920 ms"
    "Execution Time: 0.261 ms"
    
    *Объясните результат:*
    Использование индексов по партициям дает меньшее ускорение, т.к. запрос теперь должен переключаться между индексами, подбирая подходящий.


12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_part_1_title_idx;
    DROP INDEX t_books_part_2_title_idx;
    DROP INDEX t_books_part_3_title_idx;
    ```
    
    *Результат:*
    Query returned successfully in 1 secs 895 msec.

14. Создайте обычный индекс по book_id:
    ```sql
    CREATE INDEX t_books_part_idx ON t_books_part(book_id);
    ```
    
    *Результат:*
    Query returned successfully in 260 msec.
    
16. Выполните поиск по book_id:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books_part WHERE book_id = 11011;
    ```
    
    *План выполнения:*
    "Index Scan using t_books_part_1_book_id_idx on t_books_part_1 t_books_part  (cost=0.29..8.31 rows=1 width=32) (actual time=0.039..0.041 rows=1 loops=1)"
    "  Index Cond: (book_id = 11011)"
    "Planning Time: 0.917 ms"
    "Execution Time: 0.087 ms"
    
    *Объясните результат:*
    Используется индекс по партиции 1, так как id = 11011 на находится в 1 партиции. 
    Таблица не сканируется полностью.

17. Создайте индекс по полю is_active:
    ```sql
    CREATE INDEX t_books_active_idx ON t_books(is_active);
    ```
    
    *Результат:*
    CREATE INDEX
    Query returned successfully in 239 msec.
    
19. Выполните поиск активных книг с отключенным последовательным сканированием:
    ```sql
    SET enable_seqscan = off;
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE is_active = true;
    SET enable_seqscan = on;
    ```
    
    *План выполнения:*
    Пустой план
    
    *Объясните результат:*
    [Ваше объяснение]

20. Создайте составной индекс:
    ```sql
    CREATE INDEX t_books_author_title_index ON t_books(author, title);
    ```
    
    *Результат:*
    CREATE INDEX
    
    Query returned successfully in 3 secs 678 msec.
    
22. Найдите максимальное название для каждого автора:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, MAX(title) 
    FROM t_books 
    GROUP BY author;
    ```
    
    *План выполнения:*
    "HashAggregate  (cost=3530.00..3540.00 rows=1000 width=42) (actual time=117.745..118.068 rows=1003 loops=1)"
    "  Group Key: author"
    "  Batches: 1  Memory Usage: 193kB"
    "  ->  Seq Scan on t_books  (cost=0.00..2780.00 rows=150000 width=21) (actual time=0.024..18.540 rows=150000 loops=1)"
    "Planning Time: 1.123 ms"
    "Execution Time: 118.227 ms"
    
    *Объясните результат:*
    Индекс не был использован, потому что он был создан для двух полей — автора и заголовка, а в запросе происходит группировка только по автору. В результате индекс оказался неэффективным для данного запроса

23. Выберите первых 10 авторов:
    ```sql
    EXPLAIN ANALYZE
    SELECT DISTINCT author 
    FROM t_books 
    ORDER BY author 
    LIMIT 10;
    ```
    
    *План выполнения:*
    "Limit  (cost=0.29..33.94 rows=10 width=10) (actual time=0.160..0.468 rows=10 loops=1)"
    "  ->  Unique  (cost=0.29..3365.29 rows=1000 width=10) (actual time=0.158..0.465 rows=10 loops=1)"
    "        ->  Index Only Scan Backward using t_books_desc_author_inf on t_books  (cost=0.29..2990.29 rows=150000 width=10) (actual time=0.156..0.326 rows=1341 loops=1)"
    "              Heap Fetches: 0"
    "Planning Time: 0.913 ms"
    "Execution Time: 0.492 ms"
    
    *Объясните результат:*
    Запрос использует индекс для сортировки авторов в обратном порядке, что ускоряет выполнение запроса, поскольку позволяет избежать полного сканирования таблицы.

24. Выполните поиск и сортировку:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE author LIKE 'T%'
    ORDER BY author, title;
    ```
    
    *План выполнения:*
    "Sort  (cost=3155.29..3155.33 rows=15 width=21) (actual time=64.044..64.046 rows=1 loops=1)"
    "  Sort Key: author, title"
    "  Sort Method: quicksort  Memory: 25kB"
    "  ->  Seq Scan on t_books  (cost=0.00..3155.00 rows=15 width=21) (actual time=63.777..63.896 rows=1 loops=1)"
    "        Filter: ((author)::text ~~ 'T%'::text)"
    "        Rows Removed by Filter: 149999"
    "Planning Time: 0.361 ms"
    "Execution Time: 64.078 ms"
    
    *Объясните результат:*
    Фильтрация по автору существенно сузила результат до одной строки, поэтому индекс не был использован.

25. Добавьте новую книгу:
    ```sql
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150001, 'Cookbook', 'Mr. Hide', NULL, true);
    COMMIT;
    ```
    
    *Результат:*
    WARNING:  there is no transaction in progress
    COMMIT
    
    Query returned successfully in 115 msec.


26. Создайте индекс по категории:
    ```sql
    CREATE INDEX t_books_cat_idx ON t_books(category);
    ```
    
    *Результат:*
    CREATE INDEX
    
    Query returned successfully in 294 msec.
    
28. Найдите книги без категории:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE category IS NULL;
    ```
    
    *План выполнения:*
    "Index Scan using t_books_cat_idx on t_books  (cost=0.29..8.15 rows=1 width=21) (actual time=0.073..0.075 rows=1 loops=1)"
    "  Index Cond: (category IS NULL)"
    "Planning Time: 0.463 ms"
    "Execution Time: 0.099 ms"
    
    *Объясните результат:*
    Индекс ускорил работу по фильтрации по категории

29. Создайте частичные индексы:
    ```sql
    DROP INDEX t_books_cat_idx;
    CREATE INDEX t_books_cat_null_idx ON t_books(category) WHERE category IS NULL;
    ```
    
    *Результат:*
    Query returned successfully in 13 secs 291 msec.
    
31. Повторите запрос из шага 22:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE category IS NULL;
    ```
    
    *План выполнения:*
    "Index Scan using t_books_cat_null_idx on t_books  (cost=0.12..7.98 rows=1 width=21) (actual time=0.032..0.034 rows=1 loops=1)"
    "Planning Time: 1.749 ms"
    "Execution Time: 0.053 ms"
    
    *Объясните результат:*
    В запросе был задействован новый частичный индекс, что значительно ускорило выполнение, поскольку индекс был специально создан для случаев с пустой категорией. Этот индекс менее объемный, так как охватывает только те строки таблицы, которые соответствуют заданному условию.

32. Создайте частичный уникальный индекс:
    ```sql
    CREATE UNIQUE INDEX t_books_selective_unique_idx 
    ON t_books(title) 
    WHERE category = 'Science';
    
    -- Протестируйте его
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150002, 'Unique Science Book', 'Author 1', 'Science', true);
    
    -- Попробуйте вставить дубликат
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150003, 'Unique Science Book', 'Author 2', 'Science', true);
    
    -- Но можно вставить такое же название для другой категории
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150004, 'Unique Science Book', 'Author 3', 'History', true);
    ```
    
    *Результат:*
    CREATE INDEX
    Query returned successfully in 1 secs 253 msec.

    INSERT 0 1
    Query returned successfully in 116 msec.

    ERROR:  Key (title)=(Unique Science Book) already exists.duplicate key value violates unique constraint "t_books_selective_unique_idx" 

    ERROR:  duplicate key value violates unique constraint "t_books_selective_unique_idx"
    SQL state: 23505
    Detail: Key (title)=(Unique Science Book) already exists.

    INSERT 0 1
    Query returned successfully in 112 msec.
    
    *Объясните результат:*
    В другой категории дубликат добавляется,потому что на нее не распространяется действие индекса
