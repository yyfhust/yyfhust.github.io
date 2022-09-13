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
of budget and privacy are not your concern.
To name a few: Airbyte, hightouch etc..


I personally have tried Airbyte.
1. Updating API key to grant Airbyte access
2. Create a new connection with Bigquery as source and postgres as Dst.
(do not forget to white list airbyte's IP in Cloud SQl connection config)
![airbyte1](/resources/images/post1/airbyte1.png)
3. Airbyte will auto-detected the tables in bigquery and postgres.  You can choose the tables, sync mode, replication frequency etc.
![airbyte2](/resources/images/post1/airbyte2.png)


### 3. "Free" DIY 

If budget or privacy is your concern , or you just want to have everything under your own control, 
definitely you can implement such function on your own, and integrated it into your cron schedulers. 

I will detail my approach to syncing Bigquery to Postgres and schedule the job in Apache Airflow/ Cloud Composer.


```tsql
SELECT This, [Is], A, Code, Block -- Using SSMS style syntax highlighting
    , REVERSE('abc')
FROM dbo.SomeTable s
    CROSS JOIN dbo.OtherTable o;
```

