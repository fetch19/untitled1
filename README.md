USE [имябазы];
GO

DECLARE @Table_catalog NVARCHAR(128)
DECLARE @Table_schema NVARCHAR(128)
DECLARE @Table_name NVARCHAR(128)
DECLARE @Index_Name NVARCHAR(128)

DECLARE @cmd VARCHAR(4000)

-- включение сжатия для таблиц
DECLARE TableNameCursor CURSOR
FOR
SELECT Table_catalog, Table_schema, Table_name FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_TYPE = 'BASE TABLE'
ORDER BY Table_catalog, Table_schema, Table_name

OPEN TableNameCursor

FETCH NEXT FROM TableNameCursor INTO @Table_catalog, @Table_schema, @Table_name
WHILE @@fetch_status = 0
BEGIN

PRINT @Table_catalog + '.' + @Table_schema + '.' + @Table_name
SET @cmd = 'ALTER TABLE [' + @Table_catalog + '].[' + @Table_schema + '].[' + @Table_name + '] REBUILD PARTITION = ALL WITH (DATA_COMPRESSION = PAGE, MAXDOP=0)'
EXEC (@cmd)

FETCH NEXT FROM TableNameCursor INTO @Table_catalog, @Table_schema, @Table_name
END
CLOSE TableNameCursor
DEALLOCATE TableNameCursor

-- включение сжатия для индексов
DECLARE IndexCursor CURSOR
FOR
SELECT DB_NAME(), schemas.name, tables.name, indexes.name
FROM sys.schemas as schemas inner join sys.tables as tables inner join sys.indexes as indexes on tables.object_id = indexes.object_id on schemas.schema_id = tables.schema_id
ORDER BY schemas.name, tables.name, indexes.name;

OPEN IndexCursor

FETCH NEXT FROM IndexCursor INTO @Table_catalog, @Table_schema, @Table_name, @Index_Name
WHILE @@fetch_status = 0
BEGIN

PRINT @Table_catalog + '.' + @Table_schema + '.' + @Table_name + ': ' + @Index_Name

SET @cmd = 'ALTER INDEX [' + @Index_Name + '] ON [' + @Table_catalog + '].[' + @Table_schema + '].[' + @Table_name + '] REBUILD PARTITION = ALL WITH (DATA_COMPRESSION = PAGE, MAXDOP=0, FILLFACTOR=100)'
EXEC (@cmd)

FETCH NEXT FROM IndexCursor INTO @Table_catalog, @Table_schema, @Table_name, @Index_Name
END
CLOSE IndexCursor
DEALLOCATE IndexCursor

/*-- делаем шринк базы — возвращаем свободное место на диск
SELECT @cmd=(
SELECT 'DBCC SHRINKDATABASE('+ DB_NAME() + ')'
)
EXEC (@cmd)*/
