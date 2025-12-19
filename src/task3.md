## Задание 3

1. Создайте таблицу с большим количеством данных:
    ```sql
    CREATE TABLE test_cluster AS 
    SELECT 
        generate_series(1,1000000) as id,
        CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END as category,
        md5(random()::text) as data;
    ```

2. Создайте индекс:
    ```sql
    CREATE INDEX test_cluster_cat_idx ON test_cluster(category);
    ```

3. Измерьте производительность до кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    Bitmap Heap Scan on test_cluster  (cost=59.17..7696.73 rows=5000 width=68) (actual time=18.136..103.364 rows=499706 loops=1)
    Recheck Cond: (category = 'A'::text)
    Heap Blocks: exact=8334
    ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..57.92 rows=5000 width=0) (actual time=17.011..17.011 rows=499706 loops=1)
            Index Cond: (category = 'A'::text)
    Planning Time: 0.446 ms
    Execution Time: 115.529 ms
    (7 rows)
    
    *Объясните результат:*
    Используется индекс по category, но строки с A разбросаны по таблице, поэтому читается много страниц. Из-за этого выполнение занимает заметное время.

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
    CLUSTER

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    Index Scan using test_cluster_cat_idx on test_cluster  (cost=0.42..14460.92 rows=494600 width=39) (actual time=0.082..46.834 rows=499706 loops=1)
    Index Cond: (category = 'A'::text)
    Planning Time: 1.255 ms
    Execution Time: 59.615 ms
    (4 rows)
    
    *Объясните результат:*
    После кластеризации строки с одинаковым category лежат рядом, поэтому чтение данных идёт быстрее и без bitmap-сканирования.

6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*
    До кластеризации использовался bitmap scan и чтение большого числа страниц, запрос выполнялся дольше. После кластеризации данные читаются последовательно по индексу, и время выполнения заметно сократилось.