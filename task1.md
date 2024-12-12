# Задание 1. B-tree индексы в PostgreSQL

1. Запустите БД через docker compose в ./src/docker-compose.yml:

2. Выполните запрос для поиска книги с названием 'Oracle Core' и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE title = 'Oracle Core';
   ```
   *План выполнения:*
   Для выполнения будем использовать последовательное сканирование и потом фильтрацию
   
   *Объясните результат:*
    "Seq Scan on t_books  (cost=0.00..3155.00 rows=1 width=33) (actual time=52.246..52.869 rows=1 loops=1)" "  Filter: ((title)::text = 'Oracle Core'::text)" "  Rows Removed by Filter: 149999" "Planning Time: 0.192 ms" "Execution Time: 52.891 ms"

3. Создайте B-tree индексы:
   ```sql
   CREATE INDEX t_books_title_idx ON t_books(title);
   CREATE INDEX t_books_active_idx ON t_books(is_active);
   ```
   
   *Результат:*
   CREATE INDEX

   Query returned successfully in 2 secs 605 msec.

4. Проверьте информацию о созданных индексах:
   ```sql
   SELECT schemaname, tablename, indexname, indexdef
   FROM pg_catalog.pg_indexes
   WHERE tablename = 't_books';
   ```
   
   *Результат:*
   "hchoyan"	"t_books"	"t_books_title_idx"	"CREATE INDEX t_books_title_idx ON hchoyan.t_books USING btree (title)"
   "hchoyan"	"t_books"	"t_books_active_idx"	"CREATE INDEX t_books_active_idx ON hchoyan.t_books USING btree (is_active)"

5. Обновите статистику таблицы:
   ```sql
   ANALYZE t_books;
   ```
   
   *Результат:*
   ANALYZE

   Query returned successfully in 2 secs 238 msec.

6. Выполните запрос для поиска книги 'Oracle Core' и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE title = 'Oracle Core';
   ```
   
   *План выполнения:*
   "Index Scan using t_books_title_idx on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.040..0.041 rows=1 loops=1)"
   "  Index Cond: ((title)::text = 'Oracle Core'::text)"
   "Planning Time: 0.256 ms"
   "Execution Time: 0.059 ms"
   
   *Объясните результат:*
   [Ваше объяснение]

7. Выполните запрос для поиска книги по book_id и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE book_id = 18;
   ```
   
   *План выполнения:*
   "Seq Scan on t_books  (cost=0.00..3155.00 rows=1 width=33) (actual time=0.026..15.650 rows=1 loops=1)"
   "  Filter: (book_id = 18)"
   "  Rows Removed by Filter: 149999"
   "Planning Time: 0.114 ms"
   "Execution Time: 15.673 ms"
   
   *Объясните результат:*
   Нужно снова последовательно сканировать, так как созданные индексы не повлияли на поля book_id, но  из-за того, что это первичный ключ, то поиск произошел быстро автоматически

8. Выполните запрос для поиска активных книг и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE is_active = true;
   ```
   
   *План выполнения:*
   "Seq Scan on t_books  (cost=0.00..2780.00 rows=75965 width=33) (actual time=0.010..25.468 rows=75405 loops=1)"
   "  Filter: is_active"
   "  Rows Removed by Filter: 74595"
   "Planning Time: 0.451 ms"
   "Execution Time: 29.496 ms"
   
   *Объясните результат:*
   Не дал существенного ускорения, потому что столбец is_active имеет бинарные значения.

9. Посчитайте количество строк и уникальных значений:
   ```sql
   SELECT 
       COUNT(*) as total_rows,
       COUNT(DISTINCT title) as unique_titles,
       COUNT(DISTINCT category) as unique_categories,
       COUNT(DISTINCT author) as unique_authors
   FROM t_books;
   ```
   
   *Результат:*
   150000	150000	6	1003

10. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_title_idx;
    DROP INDEX t_books_active_idx;
    ```
    
    *Результат:*
    DROP INDEX

    Query returned successfully in 176 msec.

11. Основываясь на предыдущих результатах, создайте индексы для оптимизации следующих запросов:
    a. `WHERE title = $1 AND category = $2`
    b. `WHERE title = $1`
    c. `WHERE category = $1 AND author = $2`
    d. `WHERE author = $1 AND book_id = $2`
    
    *Созданные индексы:*
    a) CREATE INDEX t_books_title_idx ON t_books(title);
    b) CREATE INDEX t_books_title_idx ON t_books(title);
    c) CREATE INDEX t_books_autor_idx ON t_books(autor);
    d) CREATE INDEX t_books_autor_idx ON t_books(autor);
    
    *Объясните ваше решение:*
    1) Так как категорий мало, индекс на категорию не даст существенного ускорения
    2) Т.к. категорий мало, индекс на категорию не даст существенного ускорения
    3) Так как категорий мало, индекс на категорию не даст существенного ускорения
    4) Так как Id уже как индекс, не делаем на него индекс, он ничего не ускорит

12. Протестируйте созданные индексы.
    *Объясните результаты:*
    Протестировав убедимся в резулататах предсталвенных выше
    
14. Выполните регистронезависимый поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*
    "Seq Scan on t_books  (cost=0.00..3155.00 rows=15 width=33) (actual time=146.432..146.433 rows=0 loops=1)"
    "  Filter: ((title)::text ~~* 'Relational%'::text)"
    "  Rows Removed by Filter: 150000"
    "Planning Time: 1.806 ms"
    "Execution Time: 146.458 ms"
    
    *Объясните результат:*
    [Ваше объяснение]

15. Создайте функциональный индекс:
    ```sql
    CREATE INDEX t_books_up_title_idx ON t_books(UPPER(title));
    ```
    
    *Результат:*
    CREATE INDEX

    Query returned successfully in 13 secs 49 msec.

16. Выполните запрос из шага 13 с использованием UPPER:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE UPPER(title) LIKE 'RELATIONAL%';
    ```
    
    *План выполнения:*
    "Seq Scan on t_books  (cost=0.00..3530.00 rows=750 width=33) (actual time=85.647..85.648 rows=0 loops=1)"
    "  Filter: (upper((title)::text) ~~ 'RELATIONAL%'::text)"
    "  Rows Removed by Filter: 150000"
    "Planning Time: 0.484 ms"
    "Execution Time: 85.670 ms"
    
    *Объясните результат:*
    [Ваше объяснение]

17. Выполните поиск подстроки:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE '%Core%';
    ```
    
    *План выполнения:*
    "Seq Scan on t_books  (cost=0.00..3155.00 rows=15 width=33) (actual time=206.486..206.580 rows=1 loops=1)"
    "  Filter: ((title)::text ~~* '%Core%'::text)"
    "  Rows Removed by Filter: 149999"
    "Planning Time: 0.267 ms"
    "Execution Time: 206.605 ms"
    
    *Объясните результат:*
    [Ваше объяснение]

18. Попробуйте удалить все индексы:
    ```sql
    DO $$ 
    DECLARE
        r RECORD;
    BEGIN
        FOR r IN (SELECT indexname FROM pg_indexes 
                  WHERE tablename = 't_books' 
                  AND indexname != 'books_pkey')
        LOOP
            EXECUTE 'DROP INDEX ' || r.indexname;
        END LOOP;
    END $$;
    ```
    
    *Результат:*
    ERROR:  must be owner of index t_books_brin_cat_idx
    CONTEXT:  SQL statement "DROP INDEX t_books_brin_cat_idx"
    PL/pgSQL function inline_code_block line 9 at EXECUTE 

    SQL state: 42501
    
    *Объясните результат:*
    Ошибка указывает на то, что текущий пользователь базы данных не является владельцем индекса, который я пытаюсь удалить

19. Создайте индекс для оптимизации суффиксного поиска:
    ```sql
    -- Вариант 1: с reverse()
    CREATE INDEX t_books_rev_title_idx ON t_books(reverse(title));
    
    -- Вариант 2: с триграммами
    CREATE EXTENSION IF NOT EXISTS pg_trgm;
    CREATE INDEX t_books_trgm_idx ON t_books USING gin (title gin_trgm_ops);
    ```
    
    *Результаты тестов:*
    первый тест: Query returned successfully in 2 secs 144 msec.

    Второй: Выдал ошибку (((
    
    *Объясните результаты:*
    [Ваше объяснение]

21. Выполните поиск по точному совпадению:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title = 'Oracle Core';
    ```
    
    *План выполнения:*
    "Index Scan using t_books_title_idx on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.065..0.067 rows=1 loops=1)"
    "  Index Cond: ((title)::text = 'Oracle Core'::text)"
    "Planning Time: 0.802 ms"
    "Execution Time: 0.086 ms"
    
    *Объясните результат:*
    Индекс с reverse() не использовался

22. Выполните поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*
    "Seq Scan on t_books  (cost=0.00..3155.00 rows=15 width=33) (actual time=147.492..147.494 rows=0 loops=1)"
    "  Filter: ((title)::text ~~* 'Relational%'::text)"
    "  Rows Removed by Filter: 150000"
    "Planning Time: 0.867 ms"
    "Execution Time: 147.523 ms"
    
    *Объясните результат:*
    [Ваше объяснение]

23. Создайте свой пример индекса с обратной сортировкой:
    ```sql
    CREATE INDEX t_books_desc_idx ON t_books(title DESC);
    ```
    
    *Тестовый запрос:*
    create index t_books_desc_author_inf ON t_books(author DESC);    
    *План выполнения:*
    Query returned successfully in 639 msec.    
    *Объясните результат:*
    [Ваше объяснение]
