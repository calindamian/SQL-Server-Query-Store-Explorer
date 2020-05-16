
# SQL Server Query Store Shiny Explorer
Shiny app allowing to explore SQL Server query store and to analyse top resource consumming queries 


## Objective

SQL Server query store is a great addition starting from SQL Server 2016.

It is basically a SQL Server “flight recorder” or “black box”, capturing a history of executed queries, query runtime execution statistics, execution plans etc. against a specific database.(https://www.sqlshack.com/sql-server-query-store-overview/)

However the exploration and visualisations available in sql management studio allows a database at once experience.

The objective of Shiny Explorer is to have a global view of the queries for all databases where query store is enabled.
The current version allows interactive exploration of query activity and visualisation of query plans. 

## Prerequisites

### SQL Server Side

  * Microsoft SQL Server Version >= 2016 (SP2-CU8)
  * Enable query store for each database you want to monitor.(see https://docs.microsoft.com/en-us/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store?view=sql-server-ver15#Enabling

### Queries to extract data

The application requires 2 csv files to be uploaded (query activity and query plans). 
The files are using ";" as column separator and have a header with col names.

The queries has to be executed in each database context where query store is enabled. You can use powershell or your tool of choice. 
I provide also a R script that allows to execute a query on multiple dbs and saves result in single file.

  * Extract query activity (you can filter out data on a start date ) and save all the results in a single file. 
<details>
<summary>activity query</summary>
    ```{sql}
    DECLARE @startdate DATETIME= '20200514';
    ;WITH cte
         AS (SELECT DB_NAME(DB_ID()) AS database_name, 
                    qsint.runtime_stats_interval_id, 
                    CAST(start_time AT TIME ZONE 'Central European Standard Time' AS DATETIME) AS start_time, 
                    CAST(end_time AT TIME ZONE 'Central European Standard Time' AS DATETIME) AS end_time, 
                    CASE
                        WHEN q.object_id = 0
                        THEN 'Ad-hoc'
                        ELSE OBJECT_NAME(q.object_id)
                    END AS [ObjectName], 
                    q.query_id, 
                    qp.plan_id, 
                    q.query_parameterization_type_desc, 
                    q.last_compile_memory_kb, 
                    q.last_compile_duration, 
                    CAST(rs.last_execution_time AT TIME ZONE 'Central Europe Standard Time' AS DATETIME) AS rs_last_execution_time, 
                    CAST(DATEADD(ms, -1 * last_duration / 1000, rs.last_execution_time) AT TIME ZONE 'Central Europe Standard Time' AS DATETIME) AS rs_last_execution_start_time, 
                    CAST(q.last_compile_start_time AT TIME ZONE 'Central Europe Standard Time' AS DATETIME) AS q_last_compile_start_time, 
                    rs.execution_type_desc, 
                    rs.count_executions, 
                    rs.last_duration,
                    CASE
                        WHEN count_executions > 0
                        THEN rs.avg_duration * count_executions
                        ELSE rs.last_duration
                    END AS total_duration, 
                    rs.avg_duration, 
                    rs.max_duration, 
                    rs.min_duration, 
                    rs.avg_cpu_time, 
                    rs.last_cpu_time, 
                    rs.max_cpu_time, 
                    rs.min_cpu_time, 
                    rs.avg_cpu_time * rs.count_executions AS total_cpu_time, 
                    rs.avg_rowcount, 
                    rs.last_rowcount, 
                    rs.max_rowcount, 
                    rs.min_rowcount, 
                    rs.avg_physical_io_reads, 
                    rs.max_physical_io_reads, 
                    rs.last_physical_io_reads, 
                    rs.min_physical_io_reads, 
                    rs.avg_physical_io_reads * rs.count_executions AS total_physical_io_reads, 
                    rs.avg_logical_io_reads, 
                    rs.last_logical_io_reads, 
                    rs.max_logical_io_reads, 
                    rs.min_logical_io_reads, 
                    rs.avg_logical_io_reads * rs.count_executions AS total_logical_io_reads, 
                    rs.avg_query_max_used_memory, 
                    rs.last_query_max_used_memory, 
                    rs.max_query_max_used_memory, 
                    rs.avg_query_max_used_memory * rs.count_executions AS total_query_max_used_memory, 
                    rs.last_dop, 
                    rs.min_logical_io_writes, 
                    rs.max_logical_io_writes, 
                    rs.last_logical_io_writes, 
                    rs.avg_logical_io_writes, 
                    rs.avg_logical_io_writes * rs.count_executions AS total_logical_io_writes ,
                    CAST(qt.query_sql_text AS VARCHAR(8000)) AS query_sql_text
             FROM sys.query_store_plan qp
                  INNER JOIN sys.query_store_query q ON qp.query_id = q.query_id
                  INNER JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
                  INNER JOIN sys.query_store_runtime_stats rs ON qp.plan_id = rs.plan_id
                  INNER JOIN sys.query_store_runtime_stats_interval qsint ON qsint.runtime_stats_interval_id = rs.runtime_stats_interval_id
             ----left join sys.query_store_wait_stats qsws   available on 2017
             WHERE is_internal_query != 1)
         SELECT *
         FROM cte
         WHERE 1 = 1
               AND start_time >= CAST(@startdate AS DATETIME);
    ```
</details>  

  * Extract query plans 
<details>
<summary>plan query</summary> 
    ```sql
      DECLARE @startdate DATETIME= '20200514';
      ;WITH cte
           AS (SELECT DB_NAME(DB_ID()) AS database_name, 
                      qp.plan_id, 
                      qp.query_id, 
                      CAST(start_time AT TIME ZONE 'Central Europe Standard Time' AS DATETIME) AS start_time,
                      CASE
                          WHEN count_executions > 0
                          THEN rs.avg_duration * count_executions * 1.0 / 1000 / 1000
                          ELSE rs.last_duration * 1.0 / 1000 / 1000
                      END AS total_duration_sec
               FROM sys.query_store_plan qp
                    INNER JOIN sys.query_store_query q ON qp.query_id = q.query_id
                    INNER JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
                    INNER JOIN sys.query_store_runtime_stats rs ON qp.plan_id = rs.plan_id
                    INNER JOIN sys.query_store_runtime_stats_interval qsint ON qsint.runtime_stats_interval_id = rs.runtime_stats_interval_id),
           cte_rank
           AS (SELECT *, 
                      ROW_NUMBER() OVER(
                      ORDER BY total_duration_sec DESC) AS rn  ---rank queries based on total_duration
               FROM cte
               WHERE start_time >= CAST('' AS DATETIME))
           SELECT query_id, 
                  plan_id, 
                  query_plan
           FROM sys.query_store_plan qp
           WHERE EXISTS
           (
               SELECT *
               FROM cte_rank
               WHERE query_id = qp.query_id
                     AND plan_id = qp.plan_id
                     AND rn <= 100  --- limit the number of rows 
           );
    ```
</details>


  * Once you got the query activity and query plans files you re pretty much ready to go. Click "Upload Files.." by selecting query activity file first and then the query plans.


#### R script to extract data

Here's a small r script I wrote that executes a query on multiple databases and extract the results in single file. 


<details>
<summary>R script</summary>

```r

library(DBI)
library(tidyverse)
library(bit64)
library(lubridate)

foreach_db = function (conn , sql , exclude_db = NULL , include_db = NULL) {
            '%ni%' <- Negate('%in%')
            tb_databases  = dbGetQuery(conn, " select name from sys.databases") %>% 
                as_tibble()
            if ( is.null(include_db))
                  include_db = tb_databases %>% pull(name) 
            tb_result =  dbGetQuery(conn, " select name from sys.databases") %>% 
                         as_tibble() %>% 
                         filter (name %ni% exclude_db &  name %in% include_db) %>%
                         mutate (name =  as.character(name) ) %>%
                                  select (name)  %>%
                                  mutate( tb_filename = map  (name , function (x){  
                                                                    tryCatch(
                                                                            {
                                                                             dbGetQuery(conn,  str_c("USE " , x))
                                                                             dbGetQuery(conn,sql )%>% 
                                                                             as_tibble()}
                                                                             , 
                                                                     error=function(theError) {
                                                                            print (str_c ("foreach_db|dbGetQuery|", x, " Error:" , theError))
                                                                            return(tibble())
                                                                        } )
                                                                     })  ) %>%
                                  unnest(tb_filename)
            tb_result
    }

#configure connection
#In order to access query store tables the login has to be granted "VIEW SERVER STATE"

conn <- dbConnect(odbc::odbc(), driver= "Sql Server" ,
                  server="Your Server",                                           
                  database ="master" ,
                  UID      = "Login",
                  PWD      = "****")

#exemple query to execute on multiple dbs. Replace the query with extract queries provided 
sql = "SELECT *
      FROM sys.tables ;"

#execute the query on multile dbs (excluding some dbs ) and save data on csv file
foreach_db (conn , sql , exclude_db = c("master", "model"  ,"msdb")   ) %>% 
write_delim ("C:\\Users\\results.csv" , append = F , delim = ";" ,col_names= T )


```
</details>

## Demo
Live demo : https://calindamian.shinyapps.io/sql_query_store_explorer/

## Contact

Author : Calin Damian

Email : damcalrom@yahoo.com

GitHub: https://github.com/calindamian
