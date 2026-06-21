# Event-Handlers
# Enterprise Incremental ETL Project (AdventureWorks2019 → DataWarehouse)

## Error Handling & Logging Framework

In enterprise projects, **error handling is mandatory**. Every package should capture:

* Package Name
* Task Name
* Table Name
* Error Message
* Error Code
* Error Description
* Rows Inserted
* Rows Updated
* Start Time
* End Time
* Duration
* Status

---

# Architecture

```text
Sequence Container_EmailAddress
│
├── Execute SQL Task
│      Get MaxLastUpdatedValue
│
├── Data Flow Task
│      Incremental Load
│
├── Execute SQL Task
│      Update Audit & Config
│
└── Event Handlers
       │
       ├── OnPreExecute
       ├── OnPostExecute
       ├── OnError
       └── OnTaskFailed
```

---

# 1. Audit Table

```sql
CREATE TABLE audit_log
(
    AuditID INT IDENTITY(1,1) PRIMARY KEY,
    PackageName VARCHAR(200),
    TaskName VARCHAR(200),
    TableName VARCHAR(200),
    RecordsInserted INT DEFAULT 0,
    RecordsUpdated INT DEFAULT 0,
    RecordsDeleted INT DEFAULT 0,
    StartTime DATETIME,
    EndTime DATETIME,
    DurationSeconds INT,
    Status VARCHAR(20),
    Message VARCHAR(500),
    CreatedDate DATETIME DEFAULT GETDATE()
)
```

---

# 2. Error Log Table

```sql
CREATE TABLE ErrorLog
(
    ErrorID INT IDENTITY(1,1),
    PackageName VARCHAR(200),
    TaskName VARCHAR(200),
    SourceName VARCHAR(200),
    ErrorCode INT,
    ErrorDescription VARCHAR(MAX),
    ExecutionTime DATETIME DEFAULT GETDATE(),
    UserName VARCHAR(100),
    MachineName VARCHAR(100),
    TableName VARCHAR(200),
    FailedSQL VARCHAR(MAX),
    Status VARCHAR(20)
)
```

---

# 3. Config Table

```sql
CREATE TABLE config_table
(
    Id INT IDENTITY,
    TableName VARCHAR(100),
    LastUpdatedColumn VARCHAR(100),
    LastUpdatedValue DATETIME
)

INSERT INTO config_table VALUES ('EmailAddress', 'ModifiedDate', '1900-01-01')

```

---

# Variables

| Variable         | Datatype |
| ---------------- | -------- |
| PackageName      | String   |
| TaskName         | String   |
| TableName        | String   |
| StartTime        | DateTime |
| EndTime          | DateTime |
| RecordsInserted  | Int32    |
| RecordsUpdated   | Int32    |
| ErrorDescription | String   |
| ErrorCode        | Int32    |
| SQLStatement     | String   |

---

# CONTROL FLOW

---

# Execute SQL Task

## MaxLastUpdatedValue

SQL

```sql
SELECT LastUpdatedValue
FROM config_table
WHERE TableName='EmailAddress'
```

ResultSet

```
Single Row
```

Result Mapping

```
0

↓

User::LastUpdatedValue
```

---

# Event Handler

## OnPreExecute

Purpose

Capture package start time

Expression

```
User::StartTime = GETDATE()
```

---

# Event Handler

## OnPostExecute

Purpose

Calculate duration

```
Duration = DATEDIFF (SECOND, StartTime, GETDATE() )
```

---

# Event Handler

## OnError

When any task fails

```
MaxLastUpdatedValue
↓
Error
↓
ErrorLog
↓
audit_log
```

---

# Execute SQL Task

Insert Error

```sql
INSERT INTO ErrorLog (PackageName, TaskName, SourceName, ErrorCode, ErrorDescription, ExecutionTime, UserName, MachineName, TableName, Status)
VALUES (?, ?, ?, ?, ?, GETDATE(), SYSTEM_USER, HOST_NAME(), ?, 'Failed')

```

---

Parameter Mapping

| Variable                 | Parameter |
| ------------------------ | --------- |
| User::PackageName        | 0         |
| System::TaskName         | 1         |
| System::SourceName       | 2         |
| System::ErrorCode        | 3         |
| System::ErrorDescription | 4         |
| User::TableName          | 5         |

---

# DATA FLOW

---

# OLE DB Source

Possible Errors

```
* Connection Failed
* Table Not Found
* Column Missing
* Timeout
* Permission Denied
```

---

Error Output

```
Redirect Row
```

instead of

```
Fail Component
```

---

Destination

```
EmailAddress_Error
```

---

# Error Table

```sql
CREATE TABLE EmailAddress_Error
(
  BusinessEntityID INT,
  EmailAddressID INT,
  EmailAddress VARCHAR(100),
  rowguid UNIQUEIDENTIFIER,
  ModifiedDate DATETIME,
  ErrorCode INT,
  ErrorColumn INT,
  ErrorDate DATETIME DEFAULT GETDATE()
)
```

---

# Lookup

General

```
Full Cache

Redirect No Match
```

---

Possible Errors

```
* Duplicate Business Keys
* NULL BusinessEntityID
* Datatype mismatch
* Connection failure
```

---

Error Output

```
Redirect Row

↓

Lookup_Error
```

---

Lookup Error Table

```sql
CREATE TABLE Lookup_Error
(
BusinessEntityID INT,
EmailAddressID INT,
EmailAddress VARCHAR(100),
ErrorCode INT,
ErrorColumn INT,
CreatedDate DATETIME DEFAULT GETDATE()
)
```

---

# Row Count

Possible Errors

```
* Variable not found
* Datatype mismatch
* Overflow
```

---

# OLE DB Destination

Possible Errors

```
* PK violation
* Duplicate Key
* Datatype mismatch
* String truncation
* NULL violation
* Identity issue
```

---

Error Output

```
Redirect Row

↓

Destination_Error
```

---

Destination_Error Table

```sql
CREATE TABLE Destination_Error
(
  BusinessEntityID INT,
  EmailAddressID INT,
  EmailAddress VARCHAR(100),
  ErrorCode INT,
  ErrorColumn INT,
  CreatedDate DATETIME DEFAULT GETDATE()
)
```

---

# UPDATE TASK

Possible Errors

```
* Deadlock
* Timeout
* Lock Escalation
* Table Missing
* Permission Denied
```

---

Use TRY CATCH

```sql
  BEGIN TRY
    BEGIN TRAN
        UPDATE D
        SET
        D.EmailAddress=S.EmailAddress,
        D.ModifiedDate=S.ModifiedDate
        FROM EmailAddress D
        INNER JOIN EmailAddress_Updated S
        ON D.BusinessEntityID=S.BusinessEntityID
    COMMIT
  END TRY

BEGIN CATCH
  ROLLBACK
  
  INSERT INTO ErrorLog (PackageName, TaskName, ErrorCode, ErrorDescription, ExecutionTime, Status)
  VALUES('IncrementalLoad.dtsx', 'Update Task', ERROR_NUMBER(), ERROR_MESSAGE(), GETDATE(), 'Failed')

END CATCH

```

---

# MERGE Error Handling

```sql
BEGIN TRY

  BEGIN TRAN
  
      MERGE EmailAddress T
      USING Stage_EmailAddress S
      ON T.BusinessEntityID=S.BusinessEntityID
      
      WHEN MATCHED THEN
      UPDATE SET
      T.EmailAddress=S.EmailAddress,
      T.ModifiedDate=S.ModifiedDate
      
      WHEN NOT MATCHED THEN
      
      INSERT (BusinessEntityID, EmailAddressID, EmailAddress, rowguid, ModifiedDate)
      VALUES (S.BusinessEntityID, S.EmailAddressID, S.EmailAddress, S.rowguid, S.ModifiedDate);
  
  COMMIT
END TRY

BEGIN CATCH
  ROLLBACK

  INSERT INTO ErrorLog (PackageName, TaskName, ErrorCode, ErrorDescription, ExecutionTime, Status)
  VALUES ('IncrementalLoad.dtsx', 'MERGE', ERROR_NUMBER(), ERROR_MESSAGE(), GETDATE(), 'Failed')
END CATCH
```

---

# Final Enterprise Package

```
Sequence Container
│
├── OnPreExecute
│
├── Execute SQL Task
│      Get LastUpdatedValue
│
├── Data Flow
│      │
│      ├── OLE DB Source
│      ├── Lookup
│      ├── Row Count
│      ├── EmailAddress_Insert
│      └── EmailAddress_Update
│
├── Execute SQL Task
│      Update Audit & Config
│
├── OnPostExecute
│
└── OnError
       │
       ├── ErrorLog
       ├── audit_log
       └── Send Mail Task (Optional)
```

---

# Enterprise Best Practices

✅ Use **Redirect Row** instead of **Fail Component**

✅ Enable **SSIS Logging (OnError, OnWarning, OnTaskFailed, OnPreExecute, OnPostExecute)**

✅ Store **every failure** in an `ErrorLog` table

✅ Use **TRY...CATCH + TRANSACTION** for UPDATE/MERGE

✅ Update `config_table` **only after a successful load**

✅ Write to `audit_log` for **Success**, **Warning**, and **Failure**

✅ Add **Send Mail Task** in the `OnError` event handler to notify support teams with package name, task name, error code, and error description.
