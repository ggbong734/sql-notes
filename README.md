# SQL Notes

## Window Functions 
Source: https://mode.com/sql-tutorial/sql-window-functions/

Uses of window functions:
- Rolling averages, counts, sums 
- Ranks,dense ranks, percentiles of each group/sub-group
- Difference between current row and preceding/following row

Syntax example: To calculate running total duration for each terminal
```
SELECT start_terminal,
       duration_seconds,
       SUM(duration_seconds) OVER
         (PARTITION BY start_terminal ORDER BY start_time)
         AS running_total
  FROM tutorial.dc_bikeshare_q1_2012
 WHERE start_time < '2012-01-08'
```
`PARTITION BY` splits the data into separate groups where the running total refreshes.

`ORDER BY` orders each partition similar to how normal ORDER BY works.

---

### Other functions 
The aggregate function `SUM(*)` can be replaced by other functions:
- To create monotonically increasing number

	**ROW_NUMBER()**
	```
	ROW_NUMBER() OVER 
		(PARTITION BY start_terminal ORDER BY start_time) 
		AS monotonically_increasing_number
	```

- To list the rank or dense rank of a row from the or within a partition. 

	**RANK()** - would give identical rows a rank of 2, then skip ranks 3 and 4
	```
	RANK() OVER 
		(PARTITION BY start_terminal ORDER BY start_time) 
		AS rank
	```

	**DENSE_RANK()** - would give all identical rows rank of 2, but next row would be 3, no rank is skipped
	```
	DENSE_RANK() OVER
		(PARTITION BY start_terminal ORDER BY start_time)
		AS dense_rank
	```

- To identify what percentile (or quartile, or any other subdivisions) a given row falls into

	**NTILE()**
	```
	NTILE(100) OVER
		(PARTITION BY start_terminal ORDER BY duration_seconds)
		AS percentile,
	NTILE(4) OVER
		(PARTITION BY start_terminal ORDER BY duration_seconds)
		AS quartile
	```
- To compare rows to preceding or following rows

	**LAG()** - pulls from previous rows
	```
	# TO calculate differences between rows
	duration_seconds - LAG(duration_seconds, 1) OVER
         (PARTITION BY start_terminal ORDER BY duration_seconds) AS difference
    ```

    **LEAD()** - pulls from following rows
    ```
    duration_seconds - LEAD(duration_seconds, 1) OVER
         (PARTITION BY start_terminal ORDER BY duration_seconds) AS difference
    ```
	
- Using window Alias
```
SELECT start_terminal,
       duration_seconds,
       NTILE(4) OVER ntile_window AS quartile,
       NTILE(100) OVER ntile_window AS percentile,
       RANK() OVER ntile_window AS rank
  FROM dataset
WINDOW grouped_window AS
         (PARTITION BY start_terminal ORDER BY duration_seconds)
 ORDER BY start_terminal, duration_seconds
 ```

 ---

## Pivot table 

Alternative to **CROSSTAB**

Sample question: Create a pivot table with counts of 'good', 'bad', 'ok' occurrences [link](https://www.codewars.com/kata/5982020284a83baf2f00001c/train/sql)
```
SELECT p.name AS name,
       COUNT(CASE WHEN d.detail = 'good' THEN 1 END) AS good,
       COUNT(CASE WHEN d.detail = 'ok' THEN 1 END) AS ok,
       COUNT(CASE WHEN d.detail = 'bad' THEN 1 END) AS bad
FROM products p 
  INNER JOIN details d ON p.id = d.product_id
GROUP BY p.name
ORDER BY p.name;
```

## Lateral joins

Source: https://medium.com/kkempin/postgresqls-lateral-join-bfd6bd0199df 

For example, we have the following table `orders`.

Table "public.orders"
   Column   |            Type             | Modifiers
------------+-----------------------------+-----------
 id         | integer                     | not null
 user_id    | integer                     |
 created_at | timestamp without time zone |

Running the following code:
```
SELECT user_id, first_order_time, next_order_time, id FROM
  (SELECT user_id, min(created_at) AS first_order_time FROM orders GROUP BY user_id) o1
  LEFT JOIN LATERAL
  (SELECT id, created_at AS next_order_time
   FROM orders
   WHERE user_id = o1.user_id AND created_at > o1.first_order_time
   ORDER BY created_at ASC LIMIT 1)
   o2 ON true;
```
The result is:

 user_id |      first_order_time      |      next_order_time       | id
---------+----------------------------+----------------------------+----
       1 | 2017-06-20 04:35:03.582895 | 2017-06-20 04:58:10.137503 |  4
       3 | 2017-06-20 04:35:10.986712 | 2017-06-20 04:58:17.905277 |  5
       2 | 2017-06-20 04:35:07.564973 |                   

We use `LEFT JOIN LATERAL` which means we iterate over each of the element from o1 and we are executing subquery.
In the subquery we are selecting ID and time of the next order for each user. We can do this by using a condition created_at > o1.first_order_time which will use first_order_time from the first query (LATERAL join gives us that opportunity).

LATERAL allows us to iterate over each of the results (record) and run subquery giving an access to that record.

Example question: https://www.codewars.com/kata/5820176255c3d23f360000a9/train/sql

## Interview questions

Given two tables. One an attendance log for every student in a school district and the other a summary table with demographics for each student in the district.

attendance_events : date | student_id | attendance
all_students : student_id | school_id | grade_level | date_of_birth | hometown

Using this data, could you answer questions like the following:
- What percent of students attend school on their birthday?
- Which grade level had the largest drop in attendance between yesterday and today?


2nd question:
```
SELECT date, grade, today, yesterday, today - yesterday diff
FROM 
    (SELECT a.date, s.grade, COUNT(a.attendance) today, 
            LAG(COUNT(a.attendance), 1) OVER (PARTITION BY grade ORDER BY date) yesterday
     FROM attendance a JOIN students s ON a.student_id = s.student_id     
     WHERE a.attendance = 'present' 
       AND a.date >= (CURRENT_DATE - INTERVAL '1 day')::date    
     GROUP BY a.date, s.grade) last_days 
WHERE date = CURRENT_DATE 
ORDER BY diff ASC 
LIMIT 1;
```

Explanation: The subquery will compute the attendance for each grade for the last two days. In addition, it uses the LAG window function to include the attendance of the same grade for the preceding day (partitions by grade and orders by date).

The outer query will select the rows for the current date and compute the difference between today's and yesterday's attendance. To find the grade with the largest drop, we order by the computed difference and use LIMIT 1 to only return the row with the largest drop (or smallest increase if there is no drop for any class).