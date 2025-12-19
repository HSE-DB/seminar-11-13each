# Задание 1: BRIN индексы и bitmap-сканирование

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

4. Создайте BRIN индекс по колонке category:
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*
   Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.024..0.024 rows=0 loops=1)
   Recheck Cond: (category IS NULL)
   ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.020..0.020 rows=0 loops=1)
         Index Cond: (category IS NULL)
   Planning Time: 5.291 ms
   Execution Time: 0.928 ms
   (6 rows)
   
   *Объясните результат:*
   Используется BRIN-индекс, сначала выбираются подходящие страницы через bitmap-scan, затем строки перепроверяются. Подходящих записей нет.

6. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

7. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*
   Bitmap Heap Scan on t_books  (cost=12.16..2355.52 rows=1 width=33) (actual time=12.028..12.030 rows=0 loops=1)
   Recheck Cond: ((category)::text = 'INDEX'::text)
   Rows Removed by Index Recheck: 150000
   Filter: ((author)::text = 'SYSTEM'::text)
   Heap Blocks: lossy=1224
   ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.16 rows=74624 width=0) (actual time=0.126..0.126 rows=12240 loops=1)
         Index Cond: ((category)::text = 'INDEX'::text)
   Planning Time: 0.325 ms
   Execution Time: 12.059 ms
   (9 rows)
   
   *Объясните результат (обратите внимание на bitmap scan):*
   Используется BRIN-индекс по category, по нему выбираются страницы, дальше идёт bitmap heap scan с перепроверкой. Условие по author проверяется уже при чтении строк, поэтому много строк отбрасывается и результат пустой.

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   Sort  (cost=3099.14..3099.15 rows=6 width=7) (actual time=29.131..29.133 rows=6 loops=1)
   Sort Key: category
   Sort Method: quicksort  Memory: 25kB
   ->  HashAggregate  (cost=3099.00..3099.06 rows=6 width=7) (actual time=29.055..29.058 rows=6 loops=1)
         Group Key: category
         Batches: 1  Memory Usage: 24kB
         ->  Seq Scan on t_books  (cost=0.00..2724.00 rows=150000 width=7) (actual time=0.018..8.277 rows=150000 loops=1)
   Planning Time: 0.224 ms
   Execution Time: 31.973 ms
   (9 rows)
   
   *Объясните результат:*
   Таблица читается целиком, уникальные категории собираются через HashAggregate, после чего результат сортируется. Индексы не используются, так как нужно обработать все строки и получить уникальные значения.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   Aggregate  (cost=3099.04..3099.05 rows=1 width=8) (actual time=11.432..11.434 rows=1 loops=1)
   ->  Seq Scan on t_books  (cost=0.00..3099.00 rows=15 width=0) (actual time=11.427..11.428 rows=0 loops=1)
         Filter: ((author)::text ~~ 'S%'::text)
         Rows Removed by Filter: 150000
   Planning Time: 4.480 ms
   Execution Time: 11.468 ms
   (6 rows)
   
   *Объясните результат:*
   Выполняется полный проход по таблице с фильтрацией по author LIKE 'S%', подходящих строк нет. Индекс не используется, поэтому считается результат после последовательного сканирования.

10. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

11. Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*
   Aggregate  (cost=3475.88..3475.89 rows=1 width=8) (actual time=33.453..33.454 rows=1 loops=1)
   ->  Seq Scan on t_books  (cost=0.00..3474.00 rows=750 width=0) (actual time=33.445..33.447 rows=1 loops=1)
         Filter: (lower((title)::text) ~~ 'o%'::text)
         Rows Removed by Filter: 149999
   Planning Time: 0.659 ms
   Execution Time: 33.496 ms
   (6 rows)
   
   *Объясните результат:*
   Несмотря на созданный индекс, используется последовательное сканирование. Планировщик решил, что из-за малой выборки и вызова LOWER() чтение всей таблицы дешевле, чем использование индекса.

12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

13. Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

14. Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
    ```
   
   *План выполнения:*
   Bitmap Heap Scan on t_books  (cost=12.16..2355.52 rows=1 width=33) (actual time=0.729..0.730 rows=0 loops=1)
   Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Rows Removed by Index Recheck: 8819
   Heap Blocks: lossy=72
   ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.16 rows=74624 width=0) (actual time=0.027..0.028 rows=720 loops=1)        
         Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Planning Time: 0.315 ms
   Execution Time: 0.766 ms
   (8 rows)
   
   *Объясните результат:*
   Используется составной BRIN-индекс сразу по category и author, поэтому отбирается меньше страниц и выполнение заметно быстрее. Перепроверка остаётся, так как BRIN индекс неточный.