---
title: Use DMVs to Determine Usage Statistics and Performance of Views
description: Use DMVs to Determine Usage Statistics and Performance of Views
manager: craigg
author: MashaMSFT
ms.author: mathoma
ms.date: 09/27/2018
ms.prod: sql
ms.reviewer: ""
ms.technology: performance
ms.topic: conceptual
---

# Use DMVs to Determine Usage Statistics and Performance of Views

This article covers methodology and scripts used to get information about the **performance of queries that use views** in a database object. The intention of these scripts is to provide indicators of use and performance of various views found within a database. 

## Sys.dm_exec_query_optimizer_info

The DMV [sys.dm_exec_query_optimizer_info](https://docs.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-exec-query-optimizer-info-transact-sql) exposes statistics about the optimizations performed by the SQL Server query optimizer. These values are cumulative and begin recording when SQL Server starts.  

The below common_table_expression (CTE) uses this  DMV to provide information about the workload, such as the percentage of queries that reference a view. The results returned by this query do not indicate a performance problem by themselves, but can expose underlying issues when combined with users' complaints of slow-performing queries. 


```SQL
WITH CTE_QO AS
(
  SELECT
    occurrence
  FROM
    sys.dm_exec_query_optimizer_info
  WHERE
    ([counter] = 'optimizations')
),
QOInfo AS
(
  SELECT
    [counter]
    ,[%] = CAST((occurrence * 100.00)/(SELECT occurrence FROM CTE_QO) AS DECIMAL(5, 2))
  FROM
    sys.dm_exec_query_optimizer_info
  WHERE
    [counter] IN ('optimizations'
                  ,'trivial plan'
                  ,'no plan'
                  ,'search 0'
                  ,'search 1'
                  ,'search 2'
                  ,'timeout'
                  ,'memory limit exceeded'
                  ,'insert stmt'
                  ,'delete stmt'
                  ,'update stmt'
                  ,'merge stmt'
                  ,'contains subquery'
                  ,'view reference'
                  ,'remote query'
                  ,'dynamic cursor request'
                  ,'fast forward cursor request'
             )
)
SELECT
  [optimizations] AS [optimizations %]
  ,[trivial plan] AS [trivial plan %]
  ,[no plan] AS [no plan %]
  ,[search 0] AS [search 0 %]
  ,[search 1] AS [search 1 %]
  ,[search 2] AS [search 2 %]
  ,[timeout] AS [timeout %]
  ,[memory limit exceeded] AS [memory limit exceeded %]
  ,[insert stmt] AS [insert stmt %]
  ,[delete stmt] AS [delete stmt]
  ,[update stmt] AS [update stmt]
  ,[merge stmt] AS [merge stmt]
  ,[contains subquery] AS [contains subquery %]
  ,[view reference] AS [view reference %]
  ,[remote query] AS [remote query %]
  ,[dynamic cursor request] AS [dynamic cursor request %]
  ,[fast forward cursor request] AS [fast forward cursor request %]
FROM
  QOInfo
PIVOT (MAX([%]) FOR [counter] 
  IN ([optimizations]
      ,[trivial plan]
      ,[no plan]
      ,[search 0]
      ,[search 1]
      ,[search 2]
      ,[timeout]
      ,[memory limit exceeded]
      ,[insert stmt]
      ,[delete stmt]
      ,[update stmt]
      ,[merge stmt]
      ,[contains subquery]
      ,[view reference]
      ,[remote query]
      ,[dynamic cursor request]
      ,[fast forward cursor request])) AS p;
GO
```
Combine the results of this query with the results of the system view [sys.views](https://docs.microsoft.com/sql/relational-databases/system-catalog-views/sys-views-transact-sql) to identify query statistics, query text, and the cached execution plan. 

## Sys.views

The below CTE provides information about the number of executions, total run time, and pages read from memory. The results can be used to identify queries that may be candidates for optimization. 
  
  >[!NOTE]
  > The results of this query can vary depending on the version of SQL Server.  


```SQL
WITH CTE_VW_STATS AS
(
  SELECT
    SCHEMA_NAME(vw.schema_id) AS schemaname
    ,vw.name AS viewname
    ,vw.object_id AS viewid
  FROM
    sys.views AS vw
  WHERE
    (vw.is_ms_shipped = 0)
  INTERSECT
  SELECT
    SCHEMA_NAME(o.schema_id) AS schemaname
    ,o.Name AS name
    ,st.objectid AS viewid
  FROM
    sys.dm_exec_cached_plans cp
  CROSS APPLY
    sys.dm_exec_sql_text(cp.plan_handle) st
  INNER JOIN
    sys.objects o ON st.[objectid] = o.[object_id]
  WHERE
    st.dbid = DB_ID()
)
SELECT
  vw.schemaname
  ,vw.viewname
  ,vw.viewid
  ,DB_NAME(t.databaseid) AS databasename
  ,t.databaseid
  ,t.*
FROM
  CTE_VW_STATS AS vw
CROSS APPLY
  (
    SELECT
      st.dbid AS databaseid
      ,st.text
      ,qp.query_plan
      ,qs.*
    FROM
      sys.dm_exec_query_stats AS qs
    CROSS APPLY
      sys.dm_exec_sql_text(qs.plan_handle) AS st
    CROSS APPLY
      sys.dm_exec_query_plan(qs.plan_handle) AS qp
    WHERE
      (CHARINDEX(vw.schemaname, st.text, 1) > 0)
      AND (st.dbid = DB_ID())
  ) AS t;
GO
```

## Sys.dmv_exec_cached_plans

The final query provides information about unused views by using the DMV [sys.dmv_exec_cached_plans](https://docs.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-exec-cached-plans-transact-sql). However, the execution plan cache is dynamic, and results can vary. As such, use this query over time to determine whether or not a view is actually being used or not. 


```SQL
SELECT
  SCHEMA_NAME(vw.schema_id) AS schemaname
  ,vw.name AS name
  ,vw.object_id AS viewid
FROM
  sys.views AS vw
WHERE
  (vw.is_ms_shipped = 0)
EXCEPT
SELECT
  SCHEMA_NAME(o.schema_id) AS schemaname
  ,o.name AS name
  ,st.objectid AS viewid
FROM
  sys.dm_exec_cached_plans cp
CROSS APPLY
  sys.dm_exec_sql_text(cp.plan_handle) st
INNER JOIN
  sys.objects o ON st.[objectid] = o.[object_id]
WHERE
  st.dbid = DB_ID();
GO
```

## Related external resources

- [DMVs for Performance Tuning (Video - SQL Saturday Pordenone)](https://www.youtube.com/watch?v=9FQaFwpt3-k)
- [DMVs for Performance Tuning (Slide e Demo - SQL Saturday Pordenone)](https://www.sqlsaturday.com/589/Sessions/Details.aspx?sid=57409)
- [SQL Server Tuning in capsule form (movie-SQL Saturday Parma)](https://vimeo.com/200980883)
- [SQL Server Tuning in a nutshell (slides and Demo-SQL Saturday Parma)](https://www.sqlsaturday.com/566/Sessions/Details.aspx?sid=53988)
- [Performance Tuning With SQL Server Dynamic Management Views](https://www.red-gate.com/library/performance-tuning-with-sql-server-dynamic-management-views)
- [The Most Prominent Wait Types of your SQL Server 2016](https://channel9.msdn.com/Blogs/MVP-Data-Platform/The-Most-Prominent-Wait-Types-of-your-SQL-Server-2016)
