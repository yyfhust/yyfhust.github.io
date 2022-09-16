---
layout: post
title: Sync BigQuery To Postgres 
tags: SQL Database 
---

## Sync BigQuery To Postgres

Recently I have been building a CRUD (Create, read, update and delete) application, 
which allows some people to verify/edit/delete/update some data in our database.
All the data is saved in Bigquery, as we do heavy ETL, and data analysis on a daily basis.
However, Bigquery is not suitable to perform CRUD transactions on few rows (even though theoretically you can!). 
In the face of such drawback, I decided to sync some Bigquery tables to Postgres(deployed in GCP Cloud SQL), and build a CRUD app on 
top of Postgres.

---

### 1. Why ?
Before detailing the solution, I would like to mention why I need to sync Bigquery to Postgres. 
To answer this question, let's first compare the difference between Bigquery and Postgres.

Bigquery is a fully-managed, serverless data warehouse, which supports ANSI SQL. It can auto-scale up to handle petabytes of data volume , 
which set you free from maintaining scalability and performance of architecture. Also, 
Bigquery is an OLAP (online analytical processing) and column-oriented database. It is very performant to use Bigquery 
to perform data analysis, ETL, and bulk DML (Data manipulation). However, Bigquery is not a transactional and relational database (such as Postgres, MySQL etc) , so it is not suitable 
to perform high-frequency transactions on a single and small sets of rows, for the following reasons.
- High throughput but long latency. It takes only a few seconds to read millions of rows, but updating/inserting/deleting/reading and single row can still take a few seconds. 
- Quotas and limits of Bigquery.  

As to the CRUD app to be built, do we need high throughput? (No!), do we need low latency? (Yes! it is crucial for a good user experience).
So we need a transactional OLTP database such as Postgres. Postgres is designed to support single-row insert/update/delete/lookup. 
And we can set up Postgres in GCP Cloud SQL, 
which is a fully-managed  relational database service with some add-on features such as auto-scaling (Yes, this is very different from traditional self-hosted RDBMS ) 



### 2. Existing available solution on the market 
There are actually lots of existing commercial products which you can leverage to implement "Bigquery and Postgres Synchronization", 
if budget and privacy are not your concern.
To name a few: Airbyte, hightouch etc..


I personally have tried Airbyte.
1. Updating API key to grant Airbyte access
2. Create a new connection with Bigquery as source and postgres as Dst.
(do not forget to white list airbyte's IP in Cloud SQl connection config)
![airbyte1](/resources/images/post1/airbyte1.png)

3. Airbyte will auto-detected the tables in bigquery and postgres.  You can choose the tables, sync mode, replication frequency etc.
<p align="center">
<img src="/resources/images/post1/airbyte2.png" alt="airbyte2" width="200"/>
</p>

### 3. DIY
We would like to have this part under our own control, and integrate it with own Airflow pipeline. 
Here I detail my solution. The logic is rather simple:
1. read from Bigquery tables using Python bigquery client connector, and convert the results into dataframe
```python
## pseudo code
from google.cloud import bigquery
client = bigquery.Client(project = project )
query_job = client.query(QUERY)
df = query_job.to_dataframe()
df = df.where(pd.notnull(df), None)
```
2. Write the dataframe to postgres table. In this step, we have to handle conflicting primary keys 
( it will fail if we do not handle the case where you write some rows whose primary key already exists in the postgres table)
 Thanks to this [asnwer](https://stackoverflow.com/a/69662582), we can add a customized method to the df.to_sql method
```python
def insert_do_nothing_on_conflicts(sqltable, conn, keys, data_iter):
    """
    Execute SQL statement inserting data, and do nothing on conflicting primary keys

    Parameters
    ----------
    sqltable : pandas.io.sql.SQLTable
    conn : sqlalchemy.engine.Engine or sqlalchemy.engine.Connection
    keys : list of str
        Column names
    data_iter : Iterable that iterates the values to be inserted
    """
    from sqlalchemy.dialects.postgresql import insert
    from sqlalchemy import table, column
    columns = []
    for c in keys:
        columns.append(column(c))

    if sqltable.schema:
        table_name = '{}.{}'.format(sqltable.schema, sqltable.name)
    else:
        table_name = sqltable.name
    mytable = table(table_name, *columns)
    insert_stmt = insert(mytable).values(list(data_iter))
    do_nothing_stmt = insert_stmt.on_conflict_do_nothing(index_elements=[primaryKey])
    conn.execute(do_nothing_stmt)
df.to_sql(table_name , if_exists='append', con= engine, index=False, method=insert_do_nothing_on_conflicts, chunksize=chunksize)
```

3. package the code into an Airflow PythonOperator and schedule it !

4. possible improvement: select only NEW rows from bigquery table rather than full-table (based on _ts_ms), when the size of table gets way too big

---

I also packaged the code as a typer app , and you can find it in this [repo](https://github.com/yyfhust/Sync_Bigquery_to_Postgres)



