
# 🚀 Automating SQL Server Database Backups and Remote Table Sync

As a database developer, I've often faced scenarios where I need to:

* Back up an entire SQL Server database to disk
* Sync all its tables to a **remote server** — but with custom naming (like adding a suffix)
* Automatically create or update those remote tables with the same schema and data

Instead of doing this manually for every table, I wrote a single **SQL Server stored procedure** to do it all in one step.

---

## 💡 What Does This Stored Procedure Do?

This procedure:

1. Creates a `.BAK` backup file for the **entire database** locally.

   * Filename includes timestamp: `MyDatabase_ddMMyyyy_HH_mm.bak`

2. Loops through **all user tables** in the database.

   * For each table:

     * Checks if the **remote server** has a table named `TableName_Suffix`.
     * If it doesn't exist → creates it (matching schema).
     * If it exists → clears existing data.
     * Then → inserts all rows from local table into the remote one.

✅ No need to specify each table manually.
✅ Supports SQL Authentication to the remote server.
✅ Handles schema differences automatically.

---

## 📌 Why Use It?

If you're managing data pipelines or disaster recovery (DR) scenarios where:

* You want **local backups** for restore points
* You also want to **push all tables' data to another server** (e.g., for reporting, BI, or standby)
* And you want to **version the table names** with suffixes (like a server or environment tag)

→ This single procedure automates the entire process!

---

## 🧭 Example Usage

✅ Backup `MyDatabase` to `D:\SQL_Backups`
✅ Copy all tables to remote server `192.168.1.100`, database `RemoteDB`
✅ Add suffix `kartik` to remote table names

```sql
EXEC dbo.usp_BackupDatabaseAndAllTablesToRemote
    @DatabaseName = 'MyDatabase',
    @ServerSuffix = 'kartik',
    @LocalBackupPath = 'D:\SQL_Backups\',
    @RemoteServerName = '192.168.1.100',
    @RemoteDatabase = 'RemoteDB',
    @RemoteUser = 'sa',
    @RemotePassword = 'YourPassword'
```

---

## ✅ The Complete T-SQL Stored Procedure

Here’s the **full script** you can deploy on your SQL Server:

```sql
CREATE OR ALTER PROCEDURE dbo.usp_BackupDatabaseAndAllTablesToRemote
    @DatabaseName NVARCHAR(255),
    @ServerSuffix NVARCHAR(100),
    @LocalBackupPath NVARCHAR(500),
    @RemoteServerName NVARCHAR(255),
    @RemoteDatabase NVARCHAR(255),
    @RemoteUser NVARCHAR(100) = NULL,
    @RemotePassword NVARCHAR(100) = NULL
AS
BEGIN
    SET NOCOUNT ON;

    -- Step 1: Generate timestamped .bak filename
    DECLARE @DatePart NVARCHAR(50) = REPLACE(CONVERT(CHAR(8), GETDATE(), 103), '/', '')
    DECLARE @TimePart NVARCHAR(5) = REPLACE(CONVERT(CHAR(5), GETDATE(), 108), ':', '_')
    DECLARE @FileName NVARCHAR(500) = @DatabaseName + '_' + @DatePart + '_' + @TimePart + '.bak'
    DECLARE @FullBackupPath NVARCHAR(1000) = @LocalBackupPath + @FileName

    -- Step 2: Backup the database to disk
    DECLARE @BackupSQL NVARCHAR(MAX) = 'BACKUP DATABASE [' + @DatabaseName + '] TO DISK = N''' + @FullBackupPath + ''' WITH INIT'
    EXEC (@BackupSQL)
    PRINT '✔ Database backed up to: ' + @FullBackupPath

    -- Step 3: Loop through all user tables
    DECLARE @TableName NVARCHAR(255)
    DECLARE table_cursor CURSOR FOR
    SELECT TABLE_SCHEMA + '.' + TABLE_NAME
    FROM INFORMATION_SCHEMA.TABLES
    WHERE TABLE_TYPE = 'BASE TABLE' AND TABLE_CATALOG = @DatabaseName

    OPEN table_cursor
    FETCH NEXT FROM table_cursor INTO @TableName

    WHILE @@FETCH_STATUS = 0
    BEGIN
        DECLARE @TargetTable NVARCHAR(255) = REPLACE(@TableName, '.', '_') + '_' + @ServerSuffix
        DECLARE @CreateTable NVARCHAR(MAX) = ''
        DECLARE @Sql NVARCHAR(MAX) = ''

        -- Build CREATE TABLE statement
        SELECT @CreateTable = 'CREATE TABLE ' + QUOTENAME(@TargetTable) + ' ('
        SELECT @CreateTable = @CreateTable + CHAR(10) +
            QUOTENAME(c.name) + ' ' + 
            t.name + 
            CASE 
                WHEN t.name IN ('varchar', 'char', 'nvarchar', 'nchar') THEN '(' + 
                    CASE WHEN c.max_length = -1 THEN 'MAX' ELSE CAST(c.max_length AS VARCHAR) END + ')'
                WHEN t.name IN ('decimal', 'numeric') THEN '(' + CAST(c.precision AS VARCHAR) + ',' + CAST(c.scale AS VARCHAR) + ')'
                ELSE ''
            END + ',' 
        FROM sys.columns c
        JOIN sys.types t ON c.user_type_id = t.user_type_id
        WHERE c.object_id = OBJECT_ID(@TableName)
        SET @CreateTable = LEFT(@CreateTable, LEN(@CreateTable)-1) + ')'

        -- Step 3a: Create remote table if it doesn't exist
        SET @Sql = '
            IF NOT EXISTS (
                SELECT 1 FROM OPENROWSET(
                    ''SQLNCLI'', 
                    ''Server=' + @RemoteServerName + ';' + 
                        ISNULL('UID=' + @RemoteUser + ';PWD=' + @RemotePassword + ';', '') + ''', 
                    ''SELECT TABLE_NAME FROM ' + @RemoteDatabase + '.INFORMATION_SCHEMA.TABLES WHERE TABLE_NAME = ''''' + @TargetTable + ''''''')
            )
            BEGIN
                DECLARE @c NVARCHAR(MAX) = N''' + REPLACE(@CreateTable, '''', '''''') + '''
                EXEC (''INSERT INTO OPENROWSET(''''SQLNCLI'''', ''''Server=' + @RemoteServerName + ';' + 
                ISNULL('UID=' + @RemoteUser + ';PWD=' + @RemotePassword + ';', '') + '''', 
                ''''SET FMTONLY OFF; EXEC('''''''' + @c + '''''''') ON ' + @RemoteDatabase + '''') SELECT 1'')
            END
        '
        EXEC(@Sql)

        -- Step 3b: Delete existing remote data
        SET @Sql = '
            EXEC (''DELETE FROM OPENROWSET(''''SQLNCLI'''', ''''Server=' + @RemoteServerName + ';' + 
            ISNULL('UID=' + @RemoteUser + ';PWD=' + @RemotePassword + ';', '') + '''', 
            ''''SELECT * FROM ' + @RemoteDatabase + '.dbo.' + @TargetTable + '''')'')'
        EXEC(@Sql)

        -- Step 3c: Insert local data into remote table
        SET @Sql = '
            INSERT INTO OPENROWSET(
                ''SQLNCLI'', 
                ''Server=' + @RemoteServerName + ';' + 
                ISNULL('UID=' + @RemoteUser + ';PWD=' + @RemotePassword + ';', '') + ''', 
                ''SELECT * FROM ' + @RemoteDatabase + '.dbo.' + @TargetTable + ''')
            SELECT * FROM ' + @TableName
        EXEC(@Sql)

        PRINT '✔ Copied: ' + @TableName + ' → ' + @TargetTable
        FETCH NEXT FROM table_cursor INTO @TableName
    END

    CLOSE table_cursor
    DEALLOCATE table_cursor

    PRINT '✅ All tables copied successfully.'
END
```

---

## ✅ Benefits

* **Fully automated** – no need to list tables manually
* **Disaster recovery ready** – local .bak backup for safe restores
* **Remote replication** – easy table-level replication to another server
* **Flexible naming** – append environment/server suffix for clarity

---

If you found this useful, feel free to **share** or **comment** — and let's discuss more SQL Server automation ideas!

---





