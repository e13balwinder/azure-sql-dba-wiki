# Azure SQL Database DBA Checks Guide

## Monitoring Active Sessions with sp_whoisactive

### Overview
sp_whoisactive is a powerful diagnostic tool for monitoring currently executing sessions in Azure SQL Database. This section covers how to use it effectively to identify and resolve blocking issues.

### Basic Usage
```sql
EXEC dbatools.sp_whoisactive 
    @find_block_leaders = 1,
    @sort_order = '[blocked_session_count] DESC'
```

### Identifying Blocking Chains
1. Look for sessions with high `blocked_session_count`
2. Examine the `blocking_session_id` column to identify the root blocker
3. Check the `sql_text` and `wait_info` columns of blocking sessions

### Resolution Steps
1. For problematic sessions:
```sql
-- Get detailed information about a specific session
EXEC dbatools.sp_whoisactive 
    @session_id = <blocking_spid>,
    @get_plans = 1,
    @get_locks = 1
```

2. To kill a blocking session if necessary:
```sql
KILL <blocking_spid>
```

## Query Store Operations

### Forcing Better Query Plans

1. Identify problematic queries:
```sql
SELECT 
    qsq.query_id,
    qsp.plan_id,
    qsp.runtime_stats_interval_id,
    qsrs.avg_duration,
    qsrs.avg_cpu_time
FROM sys.query_store_query qsq
JOIN sys.query_store_plan qsp ON qsp.query_id = qsq.query_id
JOIN sys.query_store_runtime_stats qsrs ON qsrs.plan_id = qsp.plan_id
WHERE qsq.last_execution_time > DATEADD(hour, -24, GETUTCDATE())
ORDER BY qsrs.avg_duration DESC;
```

2. Force a better performing plan:
```sql
EXEC sys.sp_query_store_force_plan 
    @query_id = <query_id>, 
    @plan_id = <plan_id>;
```

3. Verify forced plan:
```sql
SELECT 
    p.plan_id,
    p.query_id,
    p.force_failure_count,
    p.last_force_failure_reason_desc
FROM sys.query_store_plan p
WHERE is_forced_plan = 1;
```

### Removing Forced Plans
```sql
EXEC sys.sp_query_store_unforce_plan 
    @query_id = <query_id>, 
    @plan_id = <plan_id>;
```

## Index Fragmentation Management

### Checking Fragmentation Levels
```sql
SELECT 
    OBJECT_SCHEMA_NAME(ips.object_id) AS schema_name,
    OBJECT_NAME(ips.object_id) AS object_name,
    i.name AS index_name,
    ips.avg_fragmentation_in_percent,
    ips.page_count
FROM sys.dm_db_index_physical_stats(
    DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
JOIN sys.indexes i ON ips.object_id = i.object_id 
    AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 5
    AND ips.page_count > 1000
ORDER BY ips.avg_fragmentation_in_percent DESC;
```

### Fragmentation Resolution Guidelines

- **Light Fragmentation (5-30%)**: Use REORGANIZE
```sql
ALTER INDEX [index_name] ON [schema].[table] REORGANIZE;
```

- **Heavy Fragmentation (>30%)**: Use REBUILD
```sql
ALTER INDEX [index_name] ON [schema].[table] REBUILD;
```

### Automated Maintenance Script
```sql
CREATE PROCEDURE dbo.ManageIndexFragmentation
    @SchemaName NVARCHAR(128),
    @TableName NVARCHAR(128),
    @FragmentationLimit INT = 30
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @SQL NVARCHAR(MAX);
    
    SELECT 
        @SQL = STRING_AGG(
            CASE 
                WHEN avg_fragmentation_in_percent > @FragmentationLimit 
                THEN 'ALTER INDEX [' + i.name + '] ON [' + 
                     OBJECT_SCHEMA_NAME(ips.object_id) + '].[' + 
                     OBJECT_NAME(ips.object_id) + '] REBUILD;'
                WHEN avg_fragmentation_in_percent > 5 
                THEN 'ALTER INDEX [' + i.name + '] ON [' + 
                     OBJECT_SCHEMA_NAME(ips.object_id) + '].[' + 
                     OBJECT_NAME(ips.object_id) + '] REORGANIZE;'
            END,
            CHAR(13)
        )
    FROM sys.dm_db_index_physical_stats(
        DB_ID(), OBJECT_ID(@SchemaName + '.' + @TableName), 
        NULL, NULL, 'LIMITED') ips
    JOIN sys.indexes i 
        ON ips.object_id = i.object_id 
        AND ips.index_id = i.index_id
    WHERE ips.avg_fragmentation_in_percent > 5
        AND ips.page_count > 1000;

    IF @SQL IS NOT NULL
        EXEC sp_executesql @SQL;
END;
```

## Important Notes

1. Always test operations during off-peak hours
2. Monitor DTU/CPU usage while performing maintenance
3. Keep track of Query Store storage usage
4. Regular cleanup of Query Store data is recommended
5. Document any forced plans in a change log