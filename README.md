# WhereUsed Stored Procedure for SQL Server

## Introduction
"I am open to any suggestions or advice you may have. Your feedback is always welcome!" 😊

## Overview

This SQL script modifies the `[dbo].[WhereUsed]` stored procedure within the `[MES_TST]` database. The procedure is designed to analyze table relationships and dependencies by identifying where a specified column is used within various database objects, such as tables, views, stored procedures, and functions.

## Features

The procedure searches for the provided column across:
- **Tables** (basic column presence)
- **Foreign Keys** (relationships between tables)
- **Primary Keys** (structural significance)
- **Views** (column usage)
- **Stored Procedures** (parameters and inline references)
- **Functions** (inline references)
- **Join Conditions** (within views and stored procedures)

## Implementation Details

The script uses a temporary table (`@Results`) to store search results and organizes the retrieved information in a structured format. It queries multiple system views such as:
- `INFORMATION_SCHEMA.COLUMNS`
- `INFORMATION_SCHEMA.TABLE_CONSTRAINTS`
- `sys.sql_modules`
- `sys.views`
- `sys.procedures`

Finally, the procedure outputs a list of objects where the column appears, categorized by type and relationship.

## How to Use

Execute the procedure by passing a column name as an argument:

```sql
EXEC dbo.WhereUsed @ColumnName = 'YourColumnName';
```


## ***SQL SOURCE:***
```sql
USE [YOUR_DB]
GO


SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO


ALTER PROCEDURE [dbo].[WhereUsed]
    @ColumnName NVARCHAR(128)
AS
BEGIN
    SET NOCOUNT ON;

    -- Variable to store results
    DECLARE @Results TABLE (
        ObjectType NVARCHAR(128),
        SchemaName NVARCHAR(128),
        ObjectName NVARCHAR(128),
        ColumnName NVARCHAR(128),
        Relationship NVARCHAR(512),
        JoinCondition NVARCHAR(MAX),
        RelatedSchema NVARCHAR(128),
        RelatedObject NVARCHAR(128),
        RelatedColumn NVARCHAR(128),
        ReferencedIn NVARCHAR(128)
    );

    -- Search for the field in the tables
    INSERT INTO @Results (ObjectType, SchemaName, ObjectName, ColumnName, Relationship, JoinCondition, RelatedSchema, RelatedObject, RelatedColumn, ReferencedIn)
    SELECT 
        'Table' AS ObjectType,
        TABLE_SCHEMA AS SchemaName,
        TABLE_NAME AS ObjectName,
        COLUMN_NAME AS ColumnName,
        'Column' AS Relationship,
        NULL AS JoinCondition,
        NULL AS RelatedSchema,
        NULL AS RelatedObject,
        NULL AS RelatedColumn,
        TABLE_NAME AS ReferencedIn
    FROM 
        INFORMATION_SCHEMA.COLUMNS
    WHERE 
        COLUMN_NAME = @ColumnName;

    -- Search for the field in foreign keys
    INSERT INTO @Results (ObjectType, SchemaName, ObjectName, ColumnName, Relationship, JoinCondition, RelatedSchema, RelatedObject, RelatedColumn, ReferencedIn)
    SELECT 
        'Table' AS ObjectType,
        FK.TABLE_SCHEMA AS SchemaName,
        FK.TABLE_NAME AS ObjectName,
        CU.COLUMN_NAME AS ColumnName,
        'Foreign Key' AS Relationship,
        'REFERENCES ' + PK.TABLE_SCHEMA + '.' + PK.TABLE_NAME + '(' + PKCU.COLUMN_NAME + ')' AS JoinCondition,
        PK.TABLE_SCHEMA AS RelatedSchema,
        PK.TABLE_NAME AS RelatedObject,
        PKCU.COLUMN_NAME AS RelatedColumn,
        FK.TABLE_NAME AS ReferencedIn
    FROM 
        INFORMATION_SCHEMA.REFERENTIAL_CONSTRAINTS RC
    INNER JOIN INFORMATION_SCHEMA.TABLE_CONSTRAINTS FK
        ON RC.CONSTRAINT_NAME = FK.CONSTRAINT_NAME
    INNER JOIN INFORMATION_SCHEMA.TABLE_CONSTRAINTS PK
        ON RC.UNIQUE_CONSTRAINT_NAME = PK.CONSTRAINT_NAME
    INNER JOIN INFORMATION_SCHEMA.KEY_COLUMN_USAGE CU
        ON RC.CONSTRAINT_NAME = CU.CONSTRAINT_NAME
    INNER JOIN INFORMATION_SCHEMA.KEY_COLUMN_USAGE PKCU
        ON PK.CONSTRAINT_NAME = PKCU.CONSTRAINT_NAME
    WHERE 
        CU.COLUMN_NAME = @ColumnName;

    -- Search for the field in primary keys
    INSERT INTO @Results (ObjectType, SchemaName, ObjectName, ColumnName, Relationship, JoinCondition, RelatedSchema, RelatedObject, RelatedColumn, ReferencedIn)
    SELECT 
        'Table' AS ObjectType,
        TC.TABLE_SCHEMA AS SchemaName,
        TC.TABLE_NAME AS ObjectName,
        KCU.COLUMN_NAME AS ColumnName,
        'Primary Key' AS Relationship,
        NULL AS JoinCondition,
        NULL AS RelatedSchema,
        NULL AS RelatedObject,
        NULL AS RelatedColumn,
        TC.TABLE_NAME AS ReferencedIn
    FROM 
        INFORMATION_SCHEMA.TABLE_CONSTRAINTS TC
    INNER JOIN INFORMATION_SCHEMA.KEY_COLUMN_USAGE KCU
        ON TC.CONSTRAINT_NAME = KCU.CONSTRAINT_NAME
    WHERE 
        KCU.COLUMN_NAME = @ColumnName
        AND TC.CONSTRAINT_TYPE = 'PRIMARY KEY';

    -- Search for the field in views
    INSERT INTO @Results (ObjectType, SchemaName, ObjectName, ColumnName, Relationship, JoinCondition, RelatedSchema, RelatedObject, RelatedColumn, ReferencedIn)
    SELECT 
        'View' AS ObjectType,
        V.TABLE_SCHEMA AS SchemaName,
        V.TABLE_NAME AS ObjectName,
        C.COLUMN_NAME AS ColumnName,
        'Column' AS Relationship,
        NULL AS JoinCondition,
        NULL AS RelatedSchema,
        NULL AS RelatedObject,
        NULL AS RelatedColumn,
        V.TABLE_NAME AS ReferencedIn
    FROM 
        INFORMATION_SCHEMA.VIEWS V
    INNER JOIN INFORMATION_SCHEMA.COLUMNS C
        ON V.TABLE_NAME = C.TABLE_NAME
        AND V.TABLE_SCHEMA = C.TABLE_SCHEMA
    WHERE 
        C.COLUMN_NAME = @ColumnName;

    -- Search for the field in stored procedures (parameters)
    INSERT INTO @Results (ObjectType, SchemaName, ObjectName, ColumnName, Relationship, JoinCondition, RelatedSchema, RelatedObject, RelatedColumn, ReferencedIn)
    SELECT 
        'Stored Procedure' AS ObjectType,
        SPECIFIC_SCHEMA AS SchemaName,
        SPECIFIC_NAME AS ObjectName,
        @ColumnName AS ColumnName,
        'Parameter' AS Relationship,
        NULL AS JoinCondition,
        NULL AS RelatedSchema,
        NULL AS RelatedObject,
        NULL AS RelatedColumn,
        SPECIFIC_NAME AS ReferencedIn
    FROM 
        INFORMATION_SCHEMA.PARAMETERS
    WHERE 
        PARAMETER_NAME = @ColumnName;

    -- Search for the field in stored procedures (in the procedure body)
    INSERT INTO @Results (ObjectType, SchemaName, ObjectName, ColumnName, Relationship, JoinCondition, RelatedSchema, RelatedObject, RelatedColumn, ReferencedIn)
    SELECT 
        'Stored Procedure' AS ObjectType,
        OBJECT_SCHEMA_NAME(OBJECT_ID) AS SchemaName,
        OBJECT_NAME(OBJECT_ID) AS ObjectName,
        @ColumnName AS ColumnName,
        'Body' AS Relationship,
        NULL AS JoinCondition,
        NULL AS RelatedSchema,
        NULL AS RelatedObject,
        NULL AS RelatedColumn,
        OBJECT_NAME(OBJECT_ID) AS ReferencedIn
    FROM 
        sys.sql_modules
    WHERE 
        OBJECT_DEFINITION(OBJECT_ID) LIKE '%' + @ColumnName + '%';

    -- Search for the field in functions (in the function body)
    INSERT INTO @Results (ObjectType, SchemaName, ObjectName, ColumnName, Relationship, JoinCondition, RelatedSchema, RelatedObject, RelatedColumn, ReferencedIn)
    SELECT 
        'Function' AS ObjectType,
        OBJECT_SCHEMA_NAME(OBJECT_ID) AS SchemaName,
        OBJECT_NAME(OBJECT_ID) AS ObjectName,
        @ColumnName AS ColumnName,
        'Body' AS Relationship,
        NULL AS JoinCondition,
        NULL AS RelatedSchema,
        NULL AS RelatedObject,
        NULL AS RelatedColumn,
        OBJECT_NAME(OBJECT_ID) AS ReferencedIn
    FROM 
        sys.sql_modules
    WHERE 
        OBJECT_DEFINITION(OBJECT_ID) LIKE '%' + @ColumnName + '%';

    -- Search for JOIN conditions in views
    INSERT INTO @Results (ObjectType, SchemaName, ObjectName, ColumnName, Relationship, JoinCondition, RelatedSchema, RelatedObject, RelatedColumn, ReferencedIn)
    SELECT 
        'View' AS ObjectType,
        SCHEMA_NAME(V.schema_id) AS SchemaName,
        V.name AS ObjectName,
        @ColumnName AS ColumnName,
        'Join Condition' AS Relationship,
        OBJECT_DEFINITION(V.object_id) AS JoinCondition,
        NULL AS RelatedSchema,
        NULL AS RelatedObject,
        NULL AS RelatedColumn,
        V.name AS ReferencedIn
    FROM 
        sys.views V
    WHERE 
        OBJECT_DEFINITION(V.object_id) LIKE '%' + @ColumnName + '%';

    -- Search for JOIN conditions in stored procedures
    INSERT INTO @Results (ObjectType, SchemaName, ObjectName, ColumnName, Relationship, JoinCondition, RelatedSchema, RelatedObject, RelatedColumn, ReferencedIn)
    SELECT 
        'Stored Procedure' AS ObjectType,
        SCHEMA_NAME(P.schema_id) AS SchemaName,
        P.name AS ObjectName,
        @ColumnName AS ColumnName,
        'Join Condition' AS Relationship,
        OBJECT_DEFINITION(P.object_id) AS JoinCondition,
        NULL AS RelatedSchema,
        NULL AS RelatedObject,
        NULL AS RelatedColumn,
        P.name AS ReferencedIn
    FROM 
        sys.procedures P
    WHERE 
        OBJECT_DEFINITION(P.object_id) LIKE '%' + @ColumnName + '%';

    -- Return the results...
    SELECT 
        ObjectType,
        SchemaName,
        ObjectName,
        ColumnName,
        Relationship,
        JoinCondition,
        RelatedSchema,
        RelatedObject,
        RelatedColumn,
        ReferencedIn
    FROM 
        @Results
    ORDER BY 
        ObjectType, SchemaName, ObjectName, ColumnName, Relationship;
END
GO
```
