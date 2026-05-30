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
   ```text
   Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.010..0.010 rows=0 loops=1)
     Recheck Cond: (category IS NULL)
     ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.008..0.008 rows=0 loops=1)
           Index Cond: (category IS NULL)
   Planning Time: 0.235 ms
   Execution Time: 0.043 ms
   ```
   
   *Объясните результат:*
   BRIN индекс был использован через bitmap scan. Строк с `NULL` в `category` в текущих данных нет, поэтому результат пустой. Запрос быстрый, потому что индекс сразу показывает, что подходящих диапазонов страниц нет.

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
   ```text
   Bitmap Heap Scan on t_books  (cost=12.16..2349.09 rows=1 width=33) (actual time=17.469..17.470 rows=0 loops=1)
     Recheck Cond: ((category)::text = 'INDEX'::text)
     Rows Removed by Index Recheck: 150000
     Filter: ((author)::text = 'SYSTEM'::text)
     Heap Blocks: lossy=1225
     ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.16 rows=74129 width=0) (actual time=0.109..0.109 rows=12250 loops=1)
           Index Cond: ((category)::text = 'INDEX'::text)
   Planning Time: 0.883 ms
   Execution Time: 17.545 ms
   ```
   
   *Объясните результат (обратите внимание на bitmap scan):*
   План использует `Bitmap Index Scan` по BRIN индексу категории и затем `Bitmap Heap Scan`. BRIN хранит информацию не по отдельным строкам, а по диапазонам страниц, поэтому bitmap получился lossy: PostgreSQL пришлось перечитать страницы таблицы и проверить условия заново. Строк с `category = 'INDEX'` и `author = 'SYSTEM'` нет, но из-за неточности BRIN было перепроверено много строк.

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ```text
   Sort  (cost=3100.14..3100.15 rows=6 width=7) (actual time=30.570..30.571 rows=6 loops=1)
     Sort Key: category
     Sort Method: quicksort  Memory: 25kB
     ->  HashAggregate  (cost=3100.00..3100.06 rows=6 width=7) (actual time=30.488..30.490 rows=6 loops=1)
           Group Key: category
           Batches: 1  Memory Usage: 24kB
           ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=7) (actual time=0.042..8.573 rows=150000 loops=1)
   Planning Time: 1.690 ms
   Execution Time: 30.938 ms
   ```
   
   *Объясните результат:*
   Для `DISTINCT category` PostgreSQL прочитал всю таблицу последовательным сканированием, собрал уникальные значения через `HashAggregate`, а потом отсортировал их. BRIN индекс здесь не помогает, потому что запросу всё равно нужно увидеть все строки, чтобы получить полный набор категорий.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ```text
   Aggregate  (cost=3100.03..3100.05 rows=1 width=8) (actual time=12.905..12.906 rows=1 loops=1)
     ->  Seq Scan on t_books  (cost=0.00..3100.00 rows=14 width=0) (actual time=12.901..12.901 rows=0 loops=1)
           Filter: ((author)::text ~~ 'S%'::text)
           Rows Removed by Filter: 150000
   Planning Time: 1.057 ms
   Execution Time: 12.987 ms
   ```
   
   *Объясните результат:*
   Запрос выполнился последовательным сканированием. BRIN индекс по `author` не подходит для такого поиска по префиксу: данные по авторам не упорядочены так, чтобы BRIN мог эффективно отсеять диапазоны страниц. В таблице не нашлось авторов, начинающихся на `S`.

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
   ```text
   Aggregate  (cost=3476.88..3476.89 rows=1 width=8) (actual time=29.641..29.642 rows=1 loops=1)
     ->  Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=0) (actual time=29.633..29.635 rows=1 loops=1)
           Filter: (lower((title)::text) ~~ 'o%'::text)
           Rows Removed by Filter: 149999
   Planning Time: 1.070 ms
   Execution Time: 29.718 ms
   ```
   
   *Объясните результат:*
   Несмотря на функциональный индекс по `LOWER(title)`, план выбрал `Seq Scan`. Обычный btree индекс по выражению не всегда используется для `LIKE 'prefix%'`; для такого шаблона обычно нужен индекс с подходящим operator class, например `text_pattern_ops`, или другая настройка коллации. В данных нашлась одна книга с названием, начинающимся на `O` после приведения к нижнему регистру.

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
   ```text
   Bitmap Heap Scan on t_books  (cost=12.16..2349.09 rows=1 width=33) (actual time=0.896..0.896 rows=0 loops=1)
     Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
     Rows Removed by Index Recheck: 8861
     Heap Blocks: lossy=73
     ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.16 rows=74129 width=0) (actual time=0.069..0.069 rows=730 loops=1)
           Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Planning Time: 0.703 ms
   Execution Time: 0.959 ms
   ```
   
   *Объясните результат:*
   Составной BRIN индекс проверяет оба условия сразу, поэтому перепроверять пришлось меньше блоков: `lossy=73` вместо `lossy=1225`. Строк всё равно нет, но план стал заметно быстрее, потому что индекс лучше отсёк лишние диапазоны страниц.
