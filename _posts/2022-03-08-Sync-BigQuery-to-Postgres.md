---
layout: post
title: Sync BigQuery To Postgres 
tags: SQL Database 
---

## Sync BigQuery To Postgres

Recently, I have been building a CRUD (Create, Read, Update, and Delete) application, which allows users to verify, edit, delete, and update data in our database. All the data is stored in BigQuery, as we perform heavy ETL and data analysis on a daily basis. However, BigQuery is not ideal for performing CRUD transactions on a few rows (even though theoretically possible). Faced with this limitation, I decided to sync some BigQuery tables to Postgres (deployed in GCP Cloud SQL) and build a CRUD app on top of Postgres.

---

### 1. Why?

Before detailing the solution, I want to explain why it is necessary to sync BigQuery to Postgres. Let's first compare the differences between BigQuery and Postgres.

BigQuery is a fully-managed, serverless data warehouse that supports ANSI SQL. It can auto-scale to handle petabytes of data volume, freeing you from maintaining scalability and performance of the architecture. BigQuery is an OLAP (Online Analytical Processing) and column-oriented database, making it highly efficient for data analysis, ETL, and bulk DML (Data Manipulation Language) operations. However, BigQuery is not a transactional and relational database (such as Postgres, MySQL, etc.), so it is not suitable for high-frequency transactions on a single or small set of rows, for the following reasons:
- High throughput but long latency: It takes only a few seconds to read millions of rows, but updating, inserting, deleting, or reading a single row can still take a few seconds.
- Quotas and limits of BigQuery.

For the CRUD app to be built, do we need high throughput? No! Do we need low latency? Yes, it is crucial for a good user experience. Therefore, a transactional OLTP database like Postgres is needed. Postgres is designed to support single-row insert, update, delete, and lookup operations. We can set up Postgres in GCP Cloud SQL, a fully-managed relational database service with add-on features such as auto-scaling (which is very different from traditional self-hosted RDBMS).

### 2. Existing Available Solutions on the Market

There are many existing commercial products available for implementing "BigQuery and Postgres Synchronization," if budget and privacy are not your concerns. To name a few: Airbyte, Hightouch, etc.

I personally have tried Airbyte:
1. Update the API key to grant Airbyte access.
2. Create a new connection with BigQuery as the source and Postgres as the destination. (Don't forget to whitelist Airbyte's IP in Cloud SQL connection configuration).
   ![airbyte1](/resources/images/post1/airbyte1.png)

3. Airbyte will auto-detect the tables in BigQuery and Postgres. You can choose the tables, sync mode, replication frequency, etc.

   <p align="center">
     <img src="/resources/images/post1/airbyte2.png" alt="airbyte2" width="200"/>
   </p>

### 3. DIY

We would like to have this part under our own control and integrate it with our own Airflow pipeline. Here I detail my solution. The logic is rather simple:

1. Read from BigQuery tables using the Python BigQuery client connector and convert the results into a DataFrame:
   ```python
   ## Pseudo code
   from google.cloud import bigquery
   import pandas as pd

   client = bigquery.Client(project=project)
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



