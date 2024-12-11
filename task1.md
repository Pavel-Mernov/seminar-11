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
![image](https://github.com/user-attachments/assets/56a008d4-8b95-4e4d-970d-566201f1858a)

4. Создайте BRIN индекс по колонке category:
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```
![image](https://github.com/user-attachments/assets/0fdec36a-399c-4d4c-b0ca-babffff7eea3)

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*
![image](https://github.com/user-attachments/assets/0a0a9510-701f-49d3-89b5-a5f93c21a038)

   
   *Объясните результат:*
   
   Bitmap Heap Scan.

   Cost = 12..16.

   Сканирование по индексу (стоимость = 12.00)

   Итоговое время - 1.4 мсек (ожидаемое - 2.9 мсек). Очень быстро.

7. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```
![image](https://github.com/user-attachments/assets/e73e96c3-277c-4fcd-92db-ddf33c786f1f)


8. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*
   ![image](https://github.com/user-attachments/assets/ee597dc0-3d87-401a-916d-dadeee3fc186)

   
   *Объясните результат (обратите внимание на bitmap scan):*

   Bitmap scan без индекса.

   150000 записей удалено через фильтр.

   Условие фильтра: author.

   Heap Blocks. Количество = 1225.

   Index Heap Scan с условием на category.

   Время - 46 миллисекунд. Ожидалось 1.2 секунд. Много.

10. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ![image](https://github.com/user-attachments/assets/5245c36e-e2c7-4ce5-bbe8-c67c75c0c316)

   
   *Объясните результат:*

   Сортировка по категории. Метод: QuickSort.

   HashAggregate (время - 67 мсек). В рамках этого: Батчи - 1. последовательное сканирование. Время: 17 мсек.

   Итого время: 75 мсек. Ожидалось: 2.6 мсек. Долго и неэффективно.

11. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ![image](https://github.com/user-attachments/assets/f5efd249-e8d7-4398-8540-71cf83e2c898)

   
   *Объясните результат:*
   Стоимость: 3100. 

   Обычное последовательное сканирование. Задан фильтр:

   Планируемое время: 7 мсек. Фактическое время: 28 мсек. В 4 раза больше.

11. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```
![image](https://github.com/user-attachments/assets/00117b18-4266-4092-aefb-1bfd75e09cc0)


12. Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*
   ![image](https://github.com/user-attachments/assets/6bca3828-651b-4572-988b-ebad771f3c55)

   
   *Объясните результат:*
   Стоимость: 3476.

   Обычное последовательное сканирование с фильтром.

   Планируемое время: 1.496 мсек.

   Фактическое время: 96 мсек.

   Индекс не работает.

12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

![image](https://github.com/user-attachments/assets/ab7f66bc-cb50-4b53-83b0-a4d4cf8d8649)

13. Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

   ![image](https://github.com/user-attachments/assets/b3b20d8a-9e40-4ba5-99d7-a7292f4b67e6)

14. Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
    ```
   
   *План выполнения:*
   ![image](https://github.com/user-attachments/assets/4f176dc0-65f0-4810-b53e-9134aa5e680f)
   
   *Объясните результат:*
   Heap Scan на 2 поля. Heap Blocks: 73. Удаление полей по фильтру.

   Сканирование по индексу на 2 поля. 

   Итого время: 2.2 мсек. Ожидалось: 1.7 мсек. Немногим больше, чем ожидалось и лучше, чем в прошлый раз.
