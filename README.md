Есть таблица анализов Analysis:

| **ключ** | **описание**            |
|----------|:------------------------|
| an_id    | ID анализа;             |
| an_name  | название анализа;       |
| an_cost  | себестоимость анализа;  |
| an_price | розничная цена анализа; |
|an_group |  группа анализов        |

Есть таблица групп анализов Groups:

| **ключ** | **описание**            |
|----------|:------------------------|
|gr_id | ID группы;|
|gr_name | название группы;|
|gr_temp | температурный режим хранения.|

Есть таблица заказов Orders:

| **ключ** | **описание**            |
|----------|:------------------------|
| ord_id | ID заказа;|
| ord_datetime | дата и время заказа;|
| ord_an | ID анализа.|
-------------------------

## 1.	Вывести название и цену для всех анализов, которые продавались 5 февраля 2025 и всю следующую неделю.
> [!TIP]
> Выглядит очень просто, надо смержить 2 таблицы и дать интервал по колонке с датой. 
```sql
SELECT an.an_name as 'Название анализа', an.an_price as 'Цена анализа'
FROM Orders ord inner join 
    Analysis an on ord.ord_id = an.an_id 
WHERE ord.ord_datetime >= DATE "2025-02-05" AND ord.ord_datetime < DATE '2025-02-12'
 ```

## 2. Нарастающим итогом рассчитать, как увеличивалось количество проданных тестов каждый месяц каждого года с разбивкой по группе.
> [!TIP]
> Нарастающий итог - это всегда оконная функция сумма с партицией по названию. Здесь усложнено группировкой и извлечением месяца с годом, поэтому использую CTE, дабы извлечь посчитать продажи в каждом месяеце в отдельной таблице (тип функции) 
```sql
WITH month_table (
    SELECT DATE_TRUNC('month', ord.ord_datetime) AS ym, gr.gr_name, COUNT(*) as quantity
    FROM Orders ord INNER JOIN Analysis an ON ord.ord_an = an.an_id 
        INNER JOIN Groups gr on gr.gr_id = an.an_group
    GROUP BY 1, 2
    )
    
SELECT ym as 'year&month', gr_name as 'name', quantity, sum(quantity) over (PARTITION BY gr_name, ORDER BY ym) as total
FROM month_table
ORDER BY 1
 ```
## 3. Вывести список анализов с выгодой от себестоимости.
> [!TIP]
> Маржа = цена - себестоимость 
```sql
SELECT an_name as 'Название анализа', an_price - an_cost as 'Выгода'
FROM Analysis
ORDER BY 2 DESC
 ```

## 4. Определить анализ, который был наиболее всего востребован за все время.
> [!TIP]
> Надо посчитать продажи каждого анализа и оставить 1 с максимальным кол-вом.
```sql
SELECT an_name as 'Название анализа', count(*) as 'total'
FROM Orders ord INNER JOIN Analysis an ON an.an_id = ord.ord_an
GROUP BY an.an_id, an.an_name
ORDER BY total DESC
LIMIT 1
```

## 5. Вывести анализы с выгодой от себестоимости выше средней.
> [!TIP]
> Снова в функции рассчитать маржу по каждому анализу. А затем сравнивать маржу каждой продажи с avg.
```sql
WITH profits AS (
    SELECT an_id, an_name, an_price - an_cost AS profit
    FROM Analysis
)

SELECT an_name, profit
FROM profits
WHERE profit > (SELECT avg(profit) FROM profits)
ORDER BY 2 DESC
 ```

## 6. Вывести анализы, на которые нет заказов за последние 3 месяца (март-май 2025).
> [!TIP]
> Смержу анализы и заказы в период март-май 25, затем оставлю только строки, где нет подходящих заказов
```sql
SELECT an.an_name as 'Название анализа'
FROM Analysis an
LEFT JOIN Orders ord ON ord.ord_an = an.an_id  
AND ord.ord_datetime >= DATE '2025-03-01' AND ord.ord_datetime < DATE '2025-06-01'
WHERE ord.ord_id IS NULL;
 ```


## 7.	Для каждого месяца с февраля 2025 вывести разницы выручки текущего месяца и предыдущего
> [!TIP]
> Снова воспользуюсь функцией, где аггрегирую выручку после февраля 25 (урезав месяц-год). Затемв новой таблице сравню выручку по месяцам, подтягивая предыдущую строчку через LAG. 
```sql
WITH month AS (
    SELECT DATE_TRUNC('month', ord.ord_datetime) as month_date, SUM(an.an_price) AS reveneu
    FROM orders ord INNER JOIN Analysis an ON an.an_id = ord.ord_an
    WHERE ord.ord_datetime >= DATE '2025-02-01'
    GROUP BY ym
)
SELECT month_date, reveneu, 
       reveneu - LEG(reveneu) OVER (ORDER BY ) month_date as 'diff_prev'
FROM month
ORDER BY 1
 ```