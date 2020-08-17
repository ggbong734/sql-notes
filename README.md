# SQL Notes

## Window Functions 
Source: https://mode.com/sql-tutorial/sql-window-functions/

Uses of window functions:
- Rolling averages, counts, sums 
- ranks,dense ranks, percentiles of each group/sub-group
- 

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