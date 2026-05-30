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
    ```text
    Bitmap Heap Scan on test_cluster  (cost=59.17..7696.73 rows=5000 width=68) (actual time=16.472..76.032 rows=500565 loops=1)
      Recheck Cond: (category = 'A'::text)
      Heap Blocks: exact=8334
      ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..57.92 rows=5000 width=0) (actual time=15.134..15.134 rows=500565 loops=1)
            Index Cond: (category = 'A'::text)
    Planning Time: 0.998 ms
    Execution Time: 87.253 ms
    ```
    
    *Объясните результат:*
    До кластеризации строки с категорией `A` распределены по всей таблице случайно. Индекс помогает найти ссылки на строки, но затем PostgreSQL приходится читать много блоков таблицы: `Heap Blocks: exact=8334`. Поэтому выполнение занимает заметное время.

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
    ```text
    CLUSTER
    ```

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```text
    Bitmap Heap Scan on test_cluster  (cost=59.17..7668.56 rows=5000 width=68) (actual time=16.618..55.717 rows=500565 loops=1)
      Recheck Cond: (category = 'A'::text)
      Heap Blocks: exact=4172
      ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..57.92 rows=5000 width=0) (actual time=15.796..15.796 rows=500565 loops=1)
            Index Cond: (category = 'A'::text)
    Planning Time: 0.700 ms
    Execution Time: 68.496 ms
    ```
    
    *Объясните результат:*
    После `CLUSTER` таблица физически упорядочена по индексу `test_cluster_cat_idx`, поэтому строки с одинаковой категорией лежат плотнее. План остался bitmap scan, но количество прочитанных heap-блоков уменьшилось с `8334` до `4172`, поэтому запрос стал быстрее.

6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*
    До кластеризации запрос выполнялся `87.253 ms`, после кластеризации — `68.496 ms`. Улучшение получилось за счёт того, что нужные строки стали располагаться более компактно на диске: PostgreSQL прочитал примерно в два раза меньше блоков таблицы. При этом индексный этап почти не изменился, основная экономия появилась именно на чтении heap.
