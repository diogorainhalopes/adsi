---
title: Lab 03 - Indexing
enableToc: false
---
# [[Lab03.pdf|Lab 03]]

## Index Info
```sql
EXEC sp_helpindex 'Person.Person';
```
Lists any indexes on a table, including indexes created by defining unique or primary key constraints defined by a create table or alter table statement.

```sql
SELECT INDEXPROPERTY(OBJECT_ID('Person.Person'),  
							'PK_Person_BusinessEntityID',  
							'IsClustered');
```

- The function INDEXPROPERTY provides information about the properties of an index (or statistics) on a given table. The first argument is the object (table) ID, the second argument is the index (or statistics) name, and the third argument is the desired property.  
- In this case, we are retrieving the property IsClustered of an index on table Person. Because the index is on the primary key, we should expect the index to be clustered.

```sql
SELECT INDEXPROPERTY(OBJECT_ID('Person.Person'),  
							'PK_Person_BusinessEntityID',  
							'IndexDepth');
```
- The result is the depth of the B+ tree that SQL Server built for this index.  
- SQL Server uses B+ trees for on-disk indexes, and hash indexes for in-memory tables.

```sql
SELECT INDEXPROPERTY(OBJECT_ID('Person.Person'),  
							'PK_Person_BusinessEntityID',  
							'IsUnique');
```
- Because the index is on the primary key, it should be unique, i.e. there are no duplicate values in the column (or combination of columns) being indexed.  
- Therefore, the IsUnique property for this index should be 1 (true).

```sql
DBCC SHOW_STATISTICS ('Person.Person', 'PK_Person_BusinessEntityID');
```

In the Results tab, SQL Server will show three results sets:  
1) When the statistics were last updated and how many rows the table had by then.  
2) Density, which is calculated as 1 / distinct values.  
3) A histogram of values, where:  
	- RANGE_HI_KEY is the upper bound of each histogram bin;  
	- RANGE_ROWS is the number of values that fall inside the bin (excluding the upper bound);  
	- EQ_ROWS is the number of values equal to the upper bound;
	- DISTINCT_RANGE_ROWS is the number of distinct values that fall inside the bin (excluding the upper bound);  
	- AVG_RANGE_ROWS is the average number of duplicate values inside the bin (excluding the upper bound).  
	- 
Note that, for a clustered index (i.e. an index with unique values):  
- DISTINCT_RANGE_ROWS = RANGE_ROWS  
- AVG_RANGE_ROWS = 1


```sql
DBCC SHOW_STATISTICS ('Person.Person', 'IX_Person_LastName_FirstName_MiddleName');
```
In the Results tab, note the following:  
- Several density values are being presented, one for each prefix of columns.  
- The histogram shows the distribution of values only for the first column in the index.  
- DISTINCT_RANGE_ROWS ≤ RANGE_ROWS because there are repeated values.  
- AVG_RANGE_ROWS ≥ 1 for the same reason.

## Execution Plan
```sql
SET STATISTICS IO ON;  
SET STATISTICS TIME ON;
```
In the toolbar, press the button Include Actual Execution Plan
```sql
SELECT * FROM Person.Address;
```
In the **Execution plan tab**, check that the system is doing a Clustered Index Scan using the clustered index on the primary key.

Hover the mouse (or click) over the Clustered Index Scan, and a large tooltip will appear.
I has useful information regarding the operation

```sql
SELECT * FROM Person.Address WHERE AddressID = 1000;
```
In the **Execution plan** tab, check that the system is doing a **Clustered Index Seek** (note that this is different from a **Clustered Index Scan**).

Check the **logical reads**

We will try removing the index to see the impact on the query execution plan.
```sql
ALTER TABLE Person.Address DROP CONSTRAINT PK_Address_AddressID;
```
Error:
```
The constraint 'PK_Address_AddressID' is being referenced by table 'SalesOrderHeader',  foreign key constraint 'FK_SalesOrderHeader_Address_ShipToAddressID'.
```

In fact, the primary key is being referenced by foreign keys on multiple tables.  
To find those tables, write the following code:
```sql
SELECT OBJECT_NAME(fk.parent_object_id) AS [table], 
		OBJECT_NAME(fk.object_id) AS [constraint]  
FROM sys.foreign_keys AS fk  
WHERE fk.referenced_object_id = OBJECT_ID('Person.Address');
```
The system view sys.foreign_keys returns a row for each foreign key constraint in the database.  
- Here we are interested in the foreign key constraints that reference the Person.Address table.  
- The query returns the tables that contain such references.

```sql
ALTER TABLE Person.BusinessEntityAddress  
DROP CONSTRAINT FK_BusinessEntityAddress_Address_AddressID; 

ALTER TABLE Sales.SalesOrderHeader  
DROP CONSTRAINT FK_SalesOrderHeader_Address_BillToAddressID;  

ALTER TABLE Sales.SalesOrderHeader  
DROP CONSTRAINT FK_SalesOrderHeader_Address_ShipToAddressID;
```

Now trying to drop the PK again:
```sql
ALTER TABLE Person.Address DROP CONSTRAINT PK_Address_AddressID;
```
Executing the query again:
```sql
SELECT * FROM Person.Address WHERE AddressID = 1000;
```

Check the **logical reads**

In the **Execution plan** tab, check that, in the absence of the index, the system is now doing a **Table Scan** (when earlier, with the index, it was doing a **Clustered Index Seek**).

Execute the following command to re-create the primary key and its clustered index:  
```sql
ALTER TABLE Person.Address  
ADD CONSTRAINT PK_Address_AddressID PRIMARY KEY(AddressID);
```

Expand Keys, Indexes and Statistics to confirm that the primary key and its clustered index are  back.

New query:
```sql
SELECT ModifiedDate FROM Person.Address WHERE ModifiedDate = '2014-01-01'
```
In the execution plan, check that the system is going through all the records in the table by scanning the clustered index associated with the primary key.
```sql
CREATE INDEX IX_Address_ModifiedDate ON Person.Address(ModifiedDate);
```
Now re-run the new query and the execution plan again (the new index is being used)

Note that the new index is a covering index for the query, i.e. the index contains all the information needed for the query, so the query can be answered based on the index alone.

```sql
SELECT * FROM Person.Address WHERE ModifiedDate = '2014-01-01';
```
Note that system is now using two indexes:  
- It uses the non-clustered index on ModifiedDate to locate the records with the desired date.  
- It uses the clustered index on the primary key to retrieve all the columns for those records.
