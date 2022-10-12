---
layout: post
title: Be careful to write "WHERE column IN (XXX)" in SQL!
tags: SQL
---

## Be careful to write "WHERE column IN (XXX)" in SQL!

Recently I was debugging a strange bug, and finally I found the root cause after delving into the SQL scripts and manually 
running and testing each snippet of SQL code. 


### 1. Simplify and reproduce the mistake

1. Emulate two tables:
Table1: (2537 rows, and table1_id is unique)
![1](/resources/images/post6/1.png)

Table2: (7479 rows)
![2](/resources/images/post6/2.png)

Now I want to select all columns from table1, and exclude rows with table1_id also existing in table2.table1_id.

So is it naturally to write the query :

```SQL
SELECT table1.* FROM `yifan-sandbox.sandbox_dataset.table1` table1
  WHERE table1.table1_id NOT IN (SELECT DISTINCT table1_id FROM `yifan-sandbox.sandbox_dataset.table2` ) 
```
But there is no any result . 
![3](/resources/images/post6/3.png)

Maybe you will say: because all the table1_id in table1 exist in table1_id of table2. It is a good intuition , but I 
know it is not a valid answer as I am familiar with our business logic and know it is not possible.  To prove in another way,
let's run another query (change NOT IN to IN):
![4](/resources/images/post6/4.png)

There are 1292 rows in the result, which means 1292 unique table1_ids from table1 exist in table2.table1_id, 
and (2537-1292=1245)  table1_ids do not exist. The result contradicts to the first query.


### 2. Understand why 

I admit that I blurted out "It is impossible" after I saw the contradiction. I struggle to understand why and figure our the reason by
trying from emulated small tables. 

Run this query 
```SQL
WITH table1 AS
 (SELECT '1' as table1_id, "dummy" as column_2  UNION ALL
  SELECT '2', "dummy" ) ,

 table2 AS
 (SELECT 'a' as table2_id, "1" as table1_id  UNION ALL
  SELECT 'b', "3" )

SELECT table1.* FROM table1
WHERE table1.table1_id NOT IN (SELECT table1_id FROM table2)
```

And we can get one row (table1_id = 1 ) in the result as expected, as only 2 is not in (1,3) 

Then run the following query again
(add a new row to table2 : table2_id = 'b', table1_id = NULL  )
```SQL
WITH table1 AS
 (SELECT 1 as table1_id, "dummy" as column_2  UNION ALL
  SELECT 2, "dummy" ) ,

 table2 AS
 (SELECT 'a' as table2_id, 1 as table1_id  UNION ALL
  SELECT 'b', 3 UNION ALL 
  SELECT 'b', NULL          )

SELECT table1.* FROM table1
WHERE table1.table1_id NOT IN (SELECT table1_id FROM table2)
```

"No data to display": we got to reproduce the issue. It is not hard to spot the difference (a new row with table1_id = NULL) 
Yeah, this is the root cause, the query failed to work as what you expect because there is NULL in the "NOT IN ( 1,3, NULL )".

Let's try to understand why 2 ""

2 NOT IN (1,3) is equivalent to 2 != 1 and 2 != 3 -> True and True -> True . 

2 NOT IN (1,3, NULL) is equivalent to 2 != 1 and 2 != 3 and 2 != NULL -> True and True and UNKNOWN -> UNKNOWN

[Explanation](https://en.wikipedia.org/wiki/Null_(SQL)#Comparisons_with_NULL_and_the_three-valued_logic_(3VL)): 
1. The comparison (> < = != ) with NULL yields UNKNOWN. To check if a value is or is't NULL, you have to use function ISNULL().
2. True and UNKNOWN = UNKNOWN
3. True or UNKNOWN = True 

rules 1 and 2 well explains why 2 is not selected. 

To prove rule 3, let's run query (remove NOT):

```SQL
WITH table1 AS
 (SELECT 1 as table1_id, "dummy" as column_2  UNION ALL
  SELECT 2, "dummy" ) ,

 table2 AS
 (SELECT 'a' as table2_id, 1 as table1_id  UNION ALL
  SELECT 'b', 3 UNION ALL 
  SELECT 'b', NULL          )

SELECT table1.* FROM table1
WHERE table1.table1_id IN (SELECT table1_id FROM table2)
```

the result is : 1, "dummy".

In nutshell:

2 NOT IN (1,3, NULL) != True 
1 IN (1,3, NULL) = True 






