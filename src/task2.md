## Задание 2

1. Удалите старую базу данных, если есть:
    ```shell
    docker compose down
    ```

2. Поднимите базу данных из src/docker-compose.yml:
    ```shell
    docker compose down && docker compose up -d
    ```

3. Обновите статистику:
    ```sql
    ANALYZE t_books;
    ```

4. Создайте полнотекстовый индекс:
    ```sql
    CREATE INDEX t_books_fts_idx ON t_books 
    USING GIN (to_tsvector('english', title));
    ```

5. Найдите книги, содержащие слово 'expert':
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
    ```
    
    *План выполнения:*
     Bitmap Heap Scan on t_books  (cost=21.03..1335.59 rows=750 width=33) (actual time=0.034..0.034 rows=1 loops=1)
     Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
     Heap Blocks: exact=1
     ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=0.024..0.025 rows=1 loops=1)
          Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
     Planning Time: 0.467 ms
     Execution Time: 0.096 ms
     (7 rows)
    
    *Объясните результат:*
    Используется GIN полнотекстовый индекс, по нему находятся подходящие записи, затем данные перепроверяются при чтении таблицы. Найдена одна книга, запрос выполняется быстро.

6. Удалите индекс:
    ```sql
    DROP INDEX t_books_fts_idx;
    ```

7. Создайте таблицу lookup:
    ```sql
    CREATE TABLE t_lookup (
         item_key VARCHAR(10) NOT NULL,
         item_value VARCHAR(100)
    );
    ```

8. Добавьте первичный ключ:
    ```sql
    ALTER TABLE t_lookup 
    ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
    ```

9. Заполните данными:
    ```sql
    INSERT INTO t_lookup 
    SELECT 
         LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
         'Value_' || generate_series(1, 150000);
    ```

10. Создайте кластеризованную таблицу:
     ```sql
     CREATE TABLE t_lookup_clustered (
          item_key VARCHAR(10) PRIMARY KEY,
          item_value VARCHAR(100)
     );
     ```

11. Заполните её теми же данными:
     ```sql
     INSERT INTO t_lookup_clustered 
     SELECT * FROM t_lookup;
     
     CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
     ```

12. Обновите статистику:
     ```sql
     ANALYZE t_lookup;
     ANALYZE t_lookup_clustered;
     ```

13. Выполните поиск по ключу в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.028..0.030 rows=1 loops=1)
     Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 0.248 ms
     Execution Time: 0.059 ms
     (4 rows)
     
     *Объясните результат:*
     Поиск выполняется через первичный ключ, используется обычный B-tree индекс. Находится одна строка, таблица целиком не читается.

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.078..0.099 rows=1 loops=1)    
     Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 0.636 ms
     Execution Time: 0.256 ms
     (4 rows)
     
     *Объясните результат:*
     Поиск тоже выполняется по первичному ключу через индекс. Кластеризация почти не влияет, так как поиск идёт по индексу, а не по порядку хранения строк.

15. Создайте индекс по значению для обычной таблицы:
     ```sql
     CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
     ```

16. Создайте индекс по значению для кластеризованной таблицы:
     ```sql
     CREATE INDEX t_lookup_clustered_value_idx 
     ON t_lookup_clustered(item_value);
     ```

17. Выполните поиск по значению в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.042..0.043 rows=0 loops=1)
     Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 0.491 ms
     Execution Time: 0.074 ms
     (4 rows)
     
     *Объясните результат:*
     Используется индекс по item_value, выполняется точечный поиск. Подходящих строк нет, но таблица целиком не сканируется.

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.026..0.026 rows=0 loops=1)
     Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 0.357 ms
     Execution Time: 0.047 ms
     (4 rows)
     
     *Объясните результат:*
     Поиск выполняется через индекс по item_value. Кластеризация даёт небольшое улучшение за счёт более компактного чтения данных.

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*
     Оба запроса используют индекс и работают быстро. В кластеризованной таблице время выполнения немного меньше, так как строки физически расположены ближе друг к другу.