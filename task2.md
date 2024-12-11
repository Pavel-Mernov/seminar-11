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

    ![image](https://github.com/user-attachments/assets/383c6d2c-d49a-409e-9f14-edbfddf0cbbd)

4. Создайте полнотекстовый индекс:
    ```sql
    CREATE INDEX t_books_fts_idx ON t_books 
    USING GIN (to_tsvector('english', title));
    ```

![image](https://github.com/user-attachments/assets/f6af9ac0-630d-491f-86a7-e22991d40c84)

5. Найдите книги, содержащие слово 'expert':
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
    ```

    *План выполнения:*

    ![image](https://github.com/user-attachments/assets/c0db6281-dc5a-4012-ae5c-5af711e1a06d)

    
    *Объясните результат:*
    Heap Scan по индексу. Перепроверка условия. 

   Скан по индексу. Проверка условия.

   Время выполнения: 1.6 мсек. Меньше ожидаемого: 3.7 мсек.

   Кост: 1336.

5. Удалите индекс:
    ```sql
    DROP INDEX t_books_fts_idx;
    ```
    ![image](https://github.com/user-attachments/assets/ef7c54df-fbae-471c-82de-30375b116a7f)

6. Создайте таблицу lookup:
    ```sql
    CREATE TABLE t_lookup (
         item_key VARCHAR(10) NOT NULL,
         item_value VARCHAR(100)
    );
    ```
    ![image](https://github.com/user-attachments/assets/81b78eb1-5ce5-4085-9efe-7342b61c6ede)

7. Добавьте первичный ключ:
    ```sql
    ALTER TABLE t_lookup 
    ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
    ```
    ![image](https://github.com/user-attachments/assets/c0791f5a-a4ee-41b7-93e9-3c29560e3115)

8. Заполните данными:
    ```sql
    INSERT INTO t_lookup 
    SELECT 
         LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
         'Value_' || generate_series(1, 150000);
    ```
![image](https://github.com/user-attachments/assets/229a174f-4524-48c7-b331-166bf0235db5)

9. Создайте кластеризованную таблицу:
     ```sql
     CREATE TABLE t_lookup_clustered (
          item_key VARCHAR(10) PRIMARY KEY,
          item_value VARCHAR(100)
     );
     ```
     ![image](https://github.com/user-attachments/assets/47615285-97e1-4609-9b73-ba4627247066)

10. Заполните её теми же данными:
     ```sql
     INSERT INTO t_lookup_clustered 
     SELECT * FROM t_lookup;
     
     CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
     ```
![image](https://github.com/user-attachments/assets/1c910e05-3985-4c84-8939-10d5e985ac6d)

11. Обновите статистику:
     ```sql
     ANALYZE t_lookup;
     ANALYZE t_lookup_clustered;
     ```
![image](https://github.com/user-attachments/assets/1faaf661-de49-4d30-9c79-9bbbacb626c0)

12. Выполните поиск по ключу в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ![image](https://github.com/user-attachments/assets/a9397f13-489f-426a-9b17-a87def37ec16)

     
     *Объясните результат:*
     Сканирование по индексу на первичный ключ. Фильтр: ограничение на первичный ключ.

    По времени: очень быстро. Примерно в 2 раза меньше ожидаемого. (0.29 мсек против 0.11 мсек)

    Кост: <8.44. Мало.

    Индекс на первичный ключ работает эффективно.

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ![image](https://github.com/user-attachments/assets/851ce540-250c-481d-994d-f3eda4c70832)
     
     *Объясните результат:*
     Сканирование по индексу с фильтром на Перв. ключ.

    Время: меньше ожидаемого (0.14 мсек против 0.23 мсек). Очень быстро.

    Стоимость: 8.44.Тоже низкая.

    Итого: примерно то же самое, что в предыдущем пункте.

16. Создайте индекс по значению для обычной таблицы:
     ```sql
     CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
     ```
![image](https://github.com/user-attachments/assets/d2c3aea8-d2da-4c4f-b21a-d298d846ac67)

17. Создайте индекс по значению для кластеризованной таблицы:
     ```sql
     CREATE INDEX t_lookup_clustered_value_idx 
     ON t_lookup_clustered(item_value);
     ```
![image](https://github.com/user-attachments/assets/bf1c66d6-1a6b-47d0-b1ed-b979d036934c)

18. Выполните поиск по значению в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ![image](https://github.com/user-attachments/assets/9d3f1934-087c-4900-b856-eb9725050718)

     
     *Объясните результат:*
     Сканирование по обычному индексу, фильтр на поле, по которому создан индекс.

     Время: 0.08 мсек (ожидалось 0.3 мсек). Значительно меньше ожидаемого.

20. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ![image](https://github.com/user-attachments/assets/af3e7647-ece3-4f53-a46a-72cb3930044f)

     
     *Объясните результат:*
     Сканирование по индексу с фильтром на поле.

    Время: 0.084 мсек, против ожидаемого: 0.505 мсек. Меньше ожидаемого.

    Стоимость: Та же: 8.44.

22. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*
     По времени - примерно одинаково эффективно. Иногда в обычной таблице быстрее. Лучшая стоимость - одинаковая.
