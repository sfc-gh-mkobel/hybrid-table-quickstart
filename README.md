# hybrid-table-quickstart

# An Introduction to Hybrid Table
<!-- ------------------------ -->
## Overview

### The Use Case

A hybrid table is a new Snowflake table type with a number of characteristics that differentiate it from existing Snowflake tables. Hybrid tables are optimized for:
- Single-row lookups and DML operations that require very low latency.
- Enabling high concurrency for those use cases that require high QPS.

With these capabilities, hybrid tables provide improved support for transactional workloads. In a given query, you can specify both hybrid tables and standard Snowflake tables, enabling you to use your transactional and historical analytical data together in Snowflake. You can join hybrid and standard Snowflake tables, copy data between them, and execute atomic transactions across both table types.

Use Cases that may benefit from hybrid table features:

- Application State:
Applications often require persisting state data. For example, you may need to store the state of a customer session (authentication, activity, etc.) in a user-facing application, or you may want to store the state of an ETL workflow to monitor the status and avoid running the entire workflow again if the workflow fails in the middle.

- Data Serving:
There are many use cases where customers want to serve data to their applications and to partners. Examples include ML model feature stores or pre-computed game statistics served to users through an API.

- Transactional Applications:
Hybrid tables may be suitable for some transactional applications, depending on their concurrency, latency, and feature requirements.

Hybrid tables provide the lower latency and higher throughput for single-row DMLs necessary for these use cases.

In this quickstart, we will use Tasty Bytes Snowflake fictional food truck business data to simulate a data serving use case. We will use two tables:
- ORDER_HEADER table -  This table stores order metadata such as TRUCK_ID, CUSTOMER_ID, ORDER_AMOUNT, etc.
- TRUCK table -  This table stores truck metadata such as TRUCK_ID,FRANCHISE_ID,MENU_TYPE_ID, etc.


### What You’ll Learn
- The basics of hybrid tables
- How to create and use hybrid tables
- The advantages of hybrid tables over standard tables
- Hybrid table unique characteristics like Indexes, primary keys, unique and foreign keys


### What You’ll Need

To complete this quickstart, attendees need the following:
- Snowflake account with hybrid tables enabled
- Account Admin credentials which you should use to execute the quickstart
 
### Prerequisites
- Familiarity with the Snowflake Snowsight interface
- Basic experience using git
- Tasty Bytes V2 Installed on your account. If you don't have it install it by running [V2 Tasty Bytes Foundational Data Model - Setup](https://github.com/snowflakecorp/frostbytes/tree/main/Tasty%20Bytes/20%20-%20setup/V2)



## Lab 0: Lab Setup

Duration: 5 Minutes

In this part of the lab, we will setup our Snowflake account, create new worksheets, role, database structures and create a Virtual Warehouse that we will use in this lab.

### Step 0.1 Creating a Worksheet

Within Worksheets, click the "+" button in the top-right corner of Snowsight and choose "SQL Worksheet"

![Screenshot 2023-10-03 at 13 23 58](https://github.com/snowflakecorp/techup-fy24snowday-hybrid_tables/assets/132544324/f0b2535a-30ff-4b7a-a1ef-1bf2b0b580e9)

Rename the Worksheet by clicking on the auto-generated Timestamp name and inputting "Hybrid Table - QuickStart"

### Step 0.2 SetUp

#### Create lab objects

Next we will create HYBRID_QUICKSTART_ROLE role, HYBRID_QUICKSTART_WH warehouse and HYBRID_QUICKSTART_DB database.

```sql
USE ROLE ACCOUNTADMIN;
-- Create role HYBRID_QUICKSTART_ROLE
CREATE OR REPLACE ROLE HYBRID_QUICKSTART_ROLE;
GRANT ROLE HYBRID_QUICKSTART_ROLE TO ROLE ACCOUNTADMIN ;

-- grant privileges on FROSTBYTE_TASTY_BYTES objects to role HYBRID_QUICKSTART_ROLE
GRANT USAGE ON DATABASE FROSTBYTE_TASTY_BYTES TO ROLE HYBRID_QUICKSTART_ROLE;
GRANT USAGE ON SCHEMA FROSTBYTE_TASTY_BYTES.RAW_POS  TO ROLE HYBRID_QUICKSTART_ROLE;
GRANT USAGE ON SCHEMA FROSTBYTE_TASTY_BYTES.RAW_CUSTOMER  TO ROLE HYBRID_QUICKSTART_ROLE;
GRANT SELECT ON ALL TABLES IN SCHEMA FROSTBYTE_TASTY_BYTES.RAW_CUSTOMER  TO ROLE HYBRID_QUICKSTART_ROLE;
GRANT SELECT ON ALL TABLES IN SCHEMA FROSTBYTE_TASTY_BYTES.RAW_POS  TO ROLE HYBRID_QUICKSTART_ROLE;

-- Create HYBRID_QUICKSTART_WH warehouse
CREATE OR REPLACE WAREHOUSE HYBRID_QUICKSTART_WH WAREHOUSE_SIZE = XSMALL, AUTO_SUSPEND = 300, AUTO_RESUME= TRUE;
GRANT OWNERSHIP ON WAREHOUSE HYBRID_QUICKSTART_WH TO ROLE HYBRID_QUICKSTART_ROLE;
GRANT CREATE DATABASE ON ACCOUNT TO ROLE HYBRID_QUICKSTART_ROLE;

-- Use role and create HYBRID_QUICKSTART_DB database and schema.
CREATE OR REPLACE DATABASE HYBRID_QUICKSTART_DB;
GRANT OWNERSHIP ON DATABASE HYBRID_QUICKSTART_DB TO ROLE HYBRID_QUICKSTART_ROLE;
CREATE OR REPLACE SCHEMA DATA;
GRANT OWNERSHIP ON SCHEMA HYBRID_QUICKSTART_DB.DATA TO ROLE HYBRID_QUICKSTART_ROLE;

-- Use role
USE ROLE HYBRID_QUICKSTART_ROLE;

-- Set lab context use HYBRID_DB_USER_(USER_NUMBER) database and DATA schema
USE DATABASE HYBRID_QUICKSTART_DB;
USE SCHEMA DATA;
```
#### Create Hybrid Table and Bulk Load Data

You may bulk load data into hybrid tables by copying from a data stage or other tables (that is, using CTAS, COPY, or INSERT INTO … SELECT).
It is strongly recommended to bulk load data into a hybrid table using a CREATE TABLE … AS SELECT statement, as there are several optimizations which can only be applied to a data load as part of table creation. You need to define all keys, indexes, and constraints at the creation of a hybrid table. 

This DDL will create a hybrid table TRUCK using CREATE TABLE … AS SELECT statement.
Note the primary key constraint on the TRUCK_ID column.


```sql
CREATE OR REPLACE HYBRID TABLE TRUCK (
	TRUCK_ID NUMBER(38,0) NOT NULL,
	MENU_TYPE_ID NUMBER(38,0),
	PRIMARY_CITY VARCHAR(16777216),
	REGION VARCHAR(16777216),
	ISO_REGION VARCHAR(16777216),
	COUNTRY VARCHAR(16777216),
	ISO_COUNTRY_CODE VARCHAR(16777216),
	FRANCHISE_FLAG NUMBER(38,0),
	YEAR NUMBER(38,0),
	MAKE VARCHAR(16777216),
	MODEL VARCHAR(16777216),
	EV_FLAG NUMBER(38,0),
	FRANCHISE_ID NUMBER(38,0),
	TRUCK_OPENING_DATE DATE,
        TRUCK_EMAIL VARCHAR NOT NULL UNIQUE,
	primary key (TRUCK_ID) 
)
AS
SELECT 
TRUCK_ID,
MENU_TYPE_ID,
PRIMARY_CITY,
REGION,
ISO_REGION,
COUNTRY,
ISO_COUNTRY_CODE,
FRANCHISE_FLAG,
YEAR,
MAKE,
MODEL,
EV_FLAG,
FRANCHISE_ID,
TRUCK_OPENING_DATE,
CONCAT(TRUCK_ID, '_truck@email.com')
FROM 
FROSTBYTE_TASTY_BYTES.RAW_POS.TRUCK;
```

This DDL will create the structure for the ORDER_HEADER hybrid table.
Note the following:
- Primary key constraint on ORDER_ID column
- Foreign key references constraint on column TRUCK_ID from table TRUCK
- secondary indexes on column ORDER_TS

```sql
CREATE OR REPLACE HYBRID TABLE ORDER_HEADER (
	ORDER_ID NUMBER(38,0) NOT NULL,
	TRUCK_ID NUMBER(38,0),
	LOCATION_ID NUMBER(19,0),
	CUSTOMER_ID NUMBER(38,0),
	SHIFT NUMBER(38,0),
	SHIFT_START_TIME TIME(9),
	SHIFT_END_TIME TIME(9),
	ORDER_CHANNEL VARCHAR(16777216),
	ORDER_TS TIMESTAMP_NTZ(9),
	SERVED_TS VARCHAR(16777216),
	ORDER_CURRENCY VARCHAR(3),
	ORDER_AMOUNT NUMBER(38,4),
	ORDER_TAX_AMOUNT VARCHAR(16777216),
	ORDER_DISCOUNT_AMOUNT VARCHAR(16777216),
	ORDER_TOTAL NUMBER(38,4),
	ORDER_STATUS VARCHAR(16777216) DEFAULT 'INQUEUE',
	primary key (ORDER_ID),
	foreign key (TRUCK_ID) references TRUCK(TRUCK_ID) ,
	index IDX01_ORDER_TS(ORDER_TS)
    );
```

This DML will insert data into table ORDER_HEADER using INSERT INTO … SELECT statement

```sql
insert into ORDER_HEADER (
     ORDER_ID,
     TRUCK_ID,
     LOCATION_ID,
     CUSTOMER_ID,
     SHIFT_START_TIME,
     SHIFT_END_TIME,
     ORDER_CHANNEL,
     ORDER_TS,
     SERVED_TS,
     ORDER_CURRENCY,
     ORDER_AMOUNT,
     ORDER_TAX_AMOUNT,
     ORDER_DISCOUNT_AMOUNT,
     ORDER_TOTAL,
     ORDER_STATUS)
     SELECT
     ORDER_ID,
     TRUCK_ID,
     LOCATION_ID,
     CUSTOMER_ID,
     SHIFT_START_TIME,
     SHIFT_END_TIME,
     ORDER_CHANNEL,
     ORDER_TS,
     SERVED_TS,
     ORDER_CURRENCY,
     ORDER_AMOUNT,
     ORDER_TAX_AMOUNT,
     ORDER_DISCOUNT_AMOUNT,
     ORDER_TOTAL,
     ''
     from FROSTBYTE_TASTY_BYTES.RAW_POS.ORDER_HEADER limit 1000;
```

## Lab 1: Explore Data
Duration: 5 Minutes

In the previous Setup lab we created HYBRID_QUICKSTART_ROLE role, HYBRID_QUICKSTART_WH warehouse, HYBRID_QUICKSTART_DB database and schema DATA. Let's use them.
```sql
-- Lab 1
-- Set lab context
USE ROLE HYBRID_QUICKSTART_ROLE;
USE WAREHOUSE HYBRID_QUICKSTART_WH;
USE DATABASE HYBRID_QUICKSTART_DB;
USE SCHEMA DATA;
```
We also created and loaded data into the TRUCK and ORDER_HEADER hybrid tables. Now we can run a few queries and review some information to get familiar with it.

View tables properties and metadata.

```sql
SHOW TABLES LIKE '%TRUCK%';
SHOW TABLES LIKE '%ORDER_HEADER%';
```
Display information about the columns in the table. Note the primary key and unique key columns.
```sql
--Describe the columns in the table TRUCK
DESC TABLE TRUCK;
--Describe the columns in the table ORDER_HEADER
DESC TABLE ORDER_HEADER;
```

View details for all hybrid tables.

```sql
--Show all HYBRID tables
SHOW HYBRID TABLES;
```

List all the indexes in your account for which you have access privileges. Note the value of the is_unique column: for the PRIMARY KEY the value is Y and for the index the value is N.
```sql
--Show all HYBRID tables indexes
SHOW INDEXES;
```

Look at a sample of the tables.
```sql
-- Simple query to look at 10 rows of data from table TRUCK
select * from TRUCK limit 10;
-- Simple query to look at 10 rows of data from table ORDER_HEADER
select * from ORDER_HEADER limit 10;
```

## Lab 2: Unique and Foreign Keys Constraints
Duration: 10 Minutes

In this part of the lab, we will test Unique and Foreign Keys Constraints.

### Step 2.1 Unique Constraints
In this step, we will test Unique Constraint which ensures that all values in a column are different.
In table TRUCK that we created in the Setup lab we defined column TRUCK_EMAIL as NOT NULL and UNIQUE.


Display information about the columns in the table. Note the unique key value for the TRUCK_EMAIL column.

```sql
-- Lab 2
-- Set lab context
USE ROLE HYBRID_QUICKSTART_ROLE;
USE WAREHOUSE HYBRID_QUICKSTART_WH;
USE DATABASE HYBRID_QUICKSTART_DB;
USE SCHEMA DATA;

--Describe the columns in the table TRUCK
DESC TABLE TRUCK;
```

Due to the unique constraint, if we attempt to insert two records with the same email address, the statement will fail.
To test it run the following statement:

```sql
-- select one of the existing email addresses
SET TRUCK_EMAIL = (SELECT TRUCK_EMAIL FROM TRUCK LIMIT 1);
-- Since TRUCK_ID is a primary key we need to calculate a new primary key value in order not to fail on the "Primary key already exists" error.
SET MAX_TRUCK_ID = (SELECT MAX(TRUCK_ID) FROM TRUCK);
--Increment max truck_id by one
SET NEW_TRUCK_ID = $MAX_TRUCK_ID+1;
insert into TRUCK values ($NEW_TRUCK_ID,2,'Stockholm','Stockholm län','Stockholm','Sweden','SE',1,2001,'Freightliner','MT45 Utilimaster',0,276,'2020-10-01',$TRUCK_EMAIL);
```
Since we configured the column TRUCK_EMAIL in table TRUCK as UNIQUE the statement failed and we should receive the following error message:
"Duplicate key value violates unique constraint "SYS_INDEX_TRUCK_UNIQUE_TRUCK_EMAIL""

Now we will create new unique email address and insert a new record to table TRUCK:

```sql
-- Create new unique email address
SET NEW_UNIQUE_EMAIL = CONCAT($NEW_TRUCK_ID, '_truck@email.com');
insert into TRUCK values ($NEW_TRUCK_ID,2,'Stockholm','Stockholm län','Stockholm','Sweden','SE',1,2001,'Freightliner','MT45 Utilimaster',0,276,'2020-10-01',$NEW_UNIQUE_EMAIL);
```
Statement should run successfully.

### Step 2.2 Insert Foreign Keys Constraints

In this step we will test foreign key constraint.
First, we will try to insert a new record to table ORDER_HEADER with non existing truck id.
It is expected that the insert statement would fail since we will violate the TRUCK table foreign key constraint.

```sql
-- Since ORDER_ID is a primary key we need to calculate a new primary key value in order not to fail on the "Primary key already exists" error.
SET MAX_ORDER_ID = (SELECT MAX(ORDER_ID) FROM ORDER_HEADER);
--Increment max ORDER_ID by one
SET NEW_ORDER_ID = ($MAX_ORDER_ID +1);

SET NONE_EXIST_TRUCK_ID = -1;
-- Insert new record to table ORDER_HEADER
insert into ORDER_HEADER values ($NEW_ORDER_ID,$NONE_EXIST_TRUCK_ID,6090,0,0,'16:00:00','23:00:00','','2022-02-18 21:38:46.000','','USD',17.0000,'','',17.0000,'');
```
The statement should fail and we should receive the following error message:

Foreign key constraint "SYS_INDEX_ORDER_HEADER_FOREIGN_KEY_TRUCK_ID_TRUCK_TRUCK_ID" is violated.

Now we will use the new NEW_TRUCK_ID variable we used in previous step and insert a new record to table ORDER_HEADER:

```sql
-- Insert new record to table ORDER_HEADER
insert into ORDER_HEADER values ($NEW_ORDER_ID,$NEW_TRUCK_ID,6090,0,0,'16:00:00','23:00:00','','2022-02-18 21:38:46.000','','USD',17.0000,'','',17.0000,'');
```

Statement should run successfully.



### Step 2.3 Truncated Active Foreign Key Constraint

In this step, we will test that the table referenced by a foreign key constraint cannot be truncated as long as the foreign key relationship exists.
To test it run the following statement:

```sql
TRUNCATE TABLE TRUCK;
```

The statement should fail and we should receive the following error message:
"391458 (0A000): Hybrid table 'TRUCK' cannot be truncated as it is involved in active foreign key constraints."

### Step 2.4 Delete Foreign Key Constraint

In this step, we will test that a record referenced by a foreign key constraint cannot be deleted as long as the foreign key reference relationship exists.

To test it run the following statement:

```sql
DELETE FROM TRUCK WHERE TRUCK_ID = $NEW_TRUCK_ID;
```

The statement should fail and we should receive the following error message:
"Foreign keys that reference key values still exist."

To successfully delete a record referenced by a foreign key constraint, you must first delete the corresponding reference record in the ORDER_HEADER table. Only after completing this step you can proceed to delete the referenced record in the TRUCK table. To test this, execute the following statement:

```sql
DELETE FROM ORDER_HEADER WHERE ORDER_ID = $NEW_ORDER_ID;
DELETE FROM TRUCK WHERE TRUCK_ID = $NEW_TRUCK_ID;
```

Both statements should run successfully.


## Lab 3: Row-Level Locking

Duration: 10 Minutes

Locking hybrid tables unlike standard tables uses row level locking for update operations. Row Level locking allows for concurrent updates on independent records.
In this lab, we will test concurrent updates to different records.

In order to test it we will run concurrent updates on two different records in the hybrid table ORDER_HEADER. We will use the main worksheet "Hybrid Table - QuickStart" we created in lab 0 and will create a new worksheet "Hybrid Table - QuickStart session 2" to simulate a new session. From the "Hybrid Table - QuickStart" worksheet we will start a new transaction using the [BEGIN](https://docs.snowflake.com/en/sql-reference/sql/begin) statement, and run an update DML statement. Before running the [COMMIT](https://docs.snowflake.com/en/sql-reference/sql/commit) transaction statement we will open the "Hybrid Table - QuickStart session 2" worksheet and run another update DML statement. finally we will commit the open transaction.

### Step 3.1 Creating a New Worksheet

Within Worksheets, click the "+" button in the top-right corner of Snowsight and choose "SQL Worksheet"

![Screenshot 2023-10-03 at 13 23 58](https://github.com/snowflakecorp/techup-fy24snowday-hybrid_tables/assets/132544324/f0b2535a-30ff-4b7a-a1ef-1bf2b0b580e9)

Rename the Worksheet by clicking on the auto-generated Timestamp name and inputting "Hybrid Table - QuickStart session 2"

### Step 3.2 Running concurrent updates

Open "Hybrid Table - QuickStart" worksheet and then select and set MAX_ORDER_ID variable.

```sql
-- Lab 4
-- Set lab context
USE ROLE HYBRID_QUICKSTART_ROLE;
USE WAREHOUSE HYBRID_QUICKSTART_WH;
USE DATABASE HYBRID_QUICKSTART_DB;
USE SCHEMA DATA;

SET MAX_ORDER_ID = (SELECT max(order_id) from ORDER_HEADER);
SELECT $MAX_ORDER_ID;
```
Note the MAX_ORDER_ID variable value.

Start a new transaction and run the first update DML statement.
```sql
-- Begins a transaction in the current session.
BEGIN;
-- Update record
UPDATE ORDER_HEADER
SET order_status = 'COMPLETED'
WHERE order_id = $MAX_ORDER_ID;
```
Note that we didn't commit the transaction so now there is an open lock on the record WHERE order_id = $MAX_ORDER_ID.

Run [SHOW TRANSACTIONS](https://docs.snowflake.com/en/sql-reference/sql/show-transactions) statement. It is expected that the SHOW TRANSACTIONS statement would return 1 single open transaction.
```sql
-- List all running transactions
SHOW TRANSACTIONS;
```

Now open the "Hybrid Table - QuickStart session 2" worksheet and then select and set MIN_ORDER_ID variable.

```sql
-- Lab 4
-- Set lab context
USE ROLE HYBRID_QUICKSTART_ROLE;
USE WAREHOUSE HYBRID_QUICKSTART_WH;
USE DATABASE HYBRID_QUICKSTART_DB;
USE SCHEMA DATA;

SET MIN_ORDER_ID = (SELECT min(order_id) from ORDER_HEADER);
SELECT $MIN_ORDER_ID;
```
Note that the MIN_ORDER_ID variable value is different from the MAX_ORDER_ID variable value we used in the first update DML statement.


Run second update DML statement.
```sql
-- Update record
UPDATE ORDER_HEADER
SET order_status = 'COMPLETED'
WHERE order_id = $MIN_ORDER_ID;
```
The Update statement should run successfully.

Open "Hybrid Table - QuickStart" worksheet and run a commit statement to commit the open transaction.

```sql
-- Commits an open transaction in the current session.
COMMIT;
```

Run the following statement to view the updated records:
```sql
SELECT * from ORDER_HEADER where order_status = 'COMPLETED';
```

## Lab 4: Consistency 
Duration: 10 Minutes

In this lab, we will demonstrate a unique feature that shows how we can run natively, easily and effectively multi-statement operations in one consistent atomic transaction across both hybrid and standard table types. 


First, we will create a new TRUCK_STANDARD table. Afterward, we'll initiate a new transaction using the [BEGIN](https://docs.snowflake.com/en/sql-reference/sql/begin) statement, execute a multi-statement DML to insert a new truck record into both the TRUCK_HYBRID table and the TRUCK_STANDARD standard table, and finally, [COMMIT](https://docs.snowflake.com/en/sql-reference/sql/commit) the transaction.

### Step 4.1 Create Table

This DDL will create a standard table TRUCK_STANDARD using CREATE TABLE … AS SELECT statement from hybrid table TRUCK.

```sql
-- Lab 4
-- Set lab context
USE ROLE HYBRID_QUICKSTART_ROLE;
USE WAREHOUSE HYBRID_QUICKSTART_WH;
USE DATABASE HYBRID_QUICKSTART_DB;
USE SCHEMA DATA;

CREATE OR REPLACE TABLE TRUCK_STANDARD (
	TRUCK_ID NUMBER(38,0) NOT NULL,
	MENU_TYPE_ID NUMBER(38,0),
	PRIMARY_CITY VARCHAR(16777216),
	REGION VARCHAR(16777216),
	ISO_REGION VARCHAR(16777216),
	COUNTRY VARCHAR(16777216),
	ISO_COUNTRY_CODE VARCHAR(16777216),
	FRANCHISE_FLAG NUMBER(38,0),
	YEAR NUMBER(38,0),
	MAKE VARCHAR(16777216),
	MODEL VARCHAR(16777216),
	EV_FLAG NUMBER(38,0),
	FRANCHISE_ID NUMBER(38,0),
	TRUCK_OPENING_DATE DATE,
    TRUCK_EMAIL VARCHAR
)
AS
SELECT 
TRUCK_ID,
MENU_TYPE_ID,
PRIMARY_CITY,
REGION,
ISO_REGION,
COUNTRY,
ISO_COUNTRY_CODE,
FRANCHISE_FLAG,
YEAR,
MAKE,
MODEL,
EV_FLAG,
FRANCHISE_ID,
TRUCK_OPENING_DATE,
TRUCK_EMAIL
FROM 
TRUCK;
```

### Step 4.2 Run Multi Statement Transaction

Set a new truck id variable and run a multi statement transaction.


```sql
SET MAX_TRUCK_ID = (SELECT MAX(TRUCK_ID) FROM TRUCK);
--Increment max truck_id by one
SET NEW_TRUCK_ID = $MAX_TRUCK_ID+1;

-- Create new unique email address
SET NEW_UNIQUE_EMAIL = CONCAT($NEW_TRUCK_ID, '_truck@email.com');

-- Begins a transaction in the current session.
begin;
-- Insert records both to standard and hybrid table types. 
insert into TRUCK_STANDARD values ($NEW_TRUCK_ID,2,'Stockholm','Stockholm län','Stockholm','Sweden','SE',1,2001,'Freightliner','MT45 Utilimaster',0,276,'2020-10-01',$NEW_UNIQUE_EMAIL);
insert into TRUCK values ($NEW_TRUCK_ID,2,'Stockholm','Stockholm län','Stockholm','Sweden','SE',1,2001,'Freightliner','MT45 Utilimaster',0,276,'2020-10-01',$NEW_UNIQUE_EMAIL);
-- Commits an open transaction in the current session.
commit;
```

### Step 4.3 Explore Data 

Now we can run select queries to review the new inserted records.

```sql
select * from TRUCK_STANDARD where TRUCK_ID = $NEW_TRUCK_ID;
select * from TRUCK where TRUCK_ID = $NEW_TRUCK_ID;
```

## Lab 5: Hybrid Querying
Duration: 5 Minutes

In this lab, we will test the join between hybrid and standard tables. We will use the TRUCK_STANDARD table created in the previous lab and join it with the hybrid table ORDER_HEADER.

### Step 5.1 Explore Data 

In the Setup lab, we already created and loaded data into the ORDER_HEADER tables. Now we can run a few queries and review some information to get familiar with it.
```sql
-- Lab 5
-- Set lab context
USE ROLE HYBRID_QUICKSTART_ROLE;
USE WAREHOUSE HYBRID_QUICKSTART_WH;
USE DATABASE HYBRID_QUICKSTART_DB;
USE SCHEMA DATA;

-- Simple query to look at 10 rows of data from standard table FROSTBYTE_TASTY_BYTES.RAW_POS.TRUCK
select * from TRUCK_STANDARD limit 10;
-- Simple query to look at 10 rows of data from hybrid table ORDER_HEADER
select * from ORDER_HEADER limit 10;
```
### Step 5.2 Join Hybrid Table and Standard Table

In order to test the join of the hybrid table ORDER_HEADER with the standard table FROSTBYTE_TASTY_BYTES.RAW_POS.TRUCK, let's run the join statement.

```sql
-- Set ORDER_ID variable
set ORDER_ID = (select order_id from ORDER_HEADER limit 1);

-- Join tables ORDER_STATE_HYBRID and TRUCK_STANDARD
select HY.*,ST.* from ORDER_HEADER as HY join TRUCK_STANDARD as ST on HY.truck_id = ST.TRUCK_ID where HY.ORDER_ID = $ORDER_ID;
```

After executing the join statement, examine and analyze the data in the result set.


## Lab 6: Security / Governance

Duration: 10 Minutes

In this lab, we will demonstrate that the security and governance functionalities that have been applied to the standard table also exist for the hybrid table. 

### Step 6.1 Hybrid Table Access Control and User Management

[Role-based access control (RBAC)](https://docs.snowflake.com/en/user-guide/security-access-control-overview) in Snowflake for hybrid tables is the same as standard tables.
The purpose of this exercise is to give you a chance to see how you can manage access to hybrid table data in Snowflake by granting privileges to some roles.

First we will create a new HYBRID_QUICKSTART_BI_USER_ROLE role

```sql
-- Lab 6
-- Set lab context
USE ROLE HYBRID_QUICKSTART_ROLE;
USE WAREHOUSE HYBRID_QUICKSTART_WH;
USE DATABASE HYBRID_QUICKSTART_DB;
USE SCHEMA DATA;

USE ROLE ACCOUNTADMIN;
CREATE ROLE HYBRID_QUICKSTART_BI_USER_ROLE;
SET MY_USER = CURRENT_USER();
GRANT ROLE HYBRID_QUICKSTART_BI_USER_ROLE TO USER IDENTIFIER($MY_USER);
```

Then we will grant USAGE privileges to HYBRID_QUICKSTART_WH warehouse, HYBRID_QUICKSTART_DB database, and all its schemas to role HYBRID_QUICKSTART_BI_USER_ROLE

```sql
-- Use HYBRID_QUICKSTART_ROLE role
USE ROLE HYBRID_QUICKSTART_ROLE;
GRANT USAGE ON WAREHOUSE HYBRID_QUICKSTART_WH TO ROLE HYBRID_QUICKSTART_BI_USER_ROLE;
GRANT USAGE ON DATABASE HYBRID_QUICKSTART_DB TO ROLE HYBRID_QUICKSTART_BI_USER_ROLE;
GRANT USAGE ON ALL SCHEMAS IN DATABASE HYBRID_QUICKSTART_DB TO HYBRID_QUICKSTART_BI_USER_ROLE;
```

Use the newly created role and try to select some data from ORDER_STATE_HYBRID hybrid table

```sql
-- Use HYBRID_QUICKSTART_BI_USER_ROLE role
USE ROLE HYBRID_QUICKSTART_BI_USER_ROLE;
-- Use HYBRID_QUICKSTART_BI_USER_ROLE database
USE DATABASE HYBRID_QUICKSTART_DB;
USE SCHEMA DATA;

select * from ORDER_HEADER limit 10;
```
We’re not able to select any data. That’s because the role HYBRID_QUICKSTART_BI_USER_ROLE has not been granted the necessary privileges.

Use role HYBRID_QUICKSTART_ROLE and grant role HYBRID_QUICKSTART_BI_USER_ROLE select privileges on all tables in schema DATA.

```sql
-- Use HYBRID_QUICKSTART_ROLE role
USE ROLE HYBRID_QUICKSTART_ROLE;
GRANT SELECT ON ALL TABLES IN SCHEMA DATA TO ROLE HYBRID_QUICKSTART_BI_USER_ROLE;
```

Try again to select some data from ORDER_STATE_HYBRID hybrid table

```sql
-- Use HYBRID_QUICKSTART_BI_USER_ROLE role
USE ROLE HYBRID_QUICKSTART_BI_USER_ROLE;
select * from ORDER_HEADER limit 10;
```

This time it worked! This is because the HYBRID_QUICKSTART_BI_USER_ROLE role has the appropriate privileges at all levels of the hierarchy.

### Step 6.2 Hybrid Table Masking Policy

In this step, we will create a new masking policy object and apply it to the TRUCK_EMAIL column in the TRUCK hybrid table using an ALTER TABLE... ALTER COLUMN statement.

First, we will create a new masking policy.

```sql
-- Set lab context
USE ROLE HYBRID_QUICKSTART_ROLE;
USE WAREHOUSE HYBRID_QUICKSTART_WH;
USE DATABASE HYBRID_QUICKSTART_DB;
USE SCHEMA DATA;

--full column masking version, always masks
create masking policy hide_column_values as
(col_value varchar) returns varchar ->
  case
     WHEN current_role() IN ('HYBRID_QUICKSTART_ROLE') THEN col_value
    else '***MASKED***'
  end;
```

Apply the policy to the hybrid table.

```sql
-- set masking policy
alter table TRUCK modify column TRUCK_EMAIL
set masking policy hide_column_values using (TRUCK_EMAIL);
```

Since we are using the HYBRID_QUICKSTART_ROLE role column TRUCK_EMAIL should not be masked.
Run the statement to see if it has the desired effect

```sql
select * from TRUCK limit 10;
```

Since we are using HYBRID_QUICKSTART_BI_USER_ROLE role column TRUCK_EMAIL should be masked.
Run the statement to see if it has the desired effect

```sql
-- Use HYBRID_QUICKSTART_BI_USER_ROLE role
USE ROLE HYBRID_QUICKSTART_BI_USER_ROLE;
select * from TRUCK limit 10;
```

## Lab 7: Cleanup
To clean up your Snowflake environment you can run the following SQL Statements:

```sql
-- Lab 7
-- Set lab context
USE ROLE HYBRID_QUICKSTART_ROLE;
USE WAREHOUSE HYBRID_QUICKSTART_WH;
USE DATABASE HYBRID_QUICKSTART_DB;
USE SCHEMA DATA;

DROP DATABASE HYBRID_QUICKSTART_DB;
DROP WAREHOUSE HYBRID_QUICKSTART_WH;
USE ROLE ACCOUNTADMIN;
DROP ROLE HYBRID_QUICKSTART_ROLE;
DROP ROLE HYBRID_QUICKSTART_BI_USER_ROLE;
```

The last step is to manually delete "Hybrid Table - QuickStart" and "Hybrid Table - QuickStart session 2" worksheets.

## Lab 8: Conclusion & Next Steps

Having completed this quickstart you have successfully
- Created Hybrid Table and Bulk Load Data
- Explore the Data 
- Learned about Unique and Foreign Keys Constraints
- Learned about hybrid table unique row level locking
- Learned about consistency and how you can run multi-statement operation in one consistent atomic transaction across both hybrid and standard table types
- Learned about hybrid querying and how to join standard table and hybrid table
- Learned that security and governance principles apply similarly to both hybrid and standard tables


### Additional References:
- [Snowflake unistore landing page](https://www.snowflake.com/en/data-cloud/workloads/unistore/)
- [Build, Run, and Distribute Custom Apps on Snowflake Snowday 2023](https://snowflake.wistia.com/medias/mvtdzo06c5)
- [Snowflake BUILD: Simplify Application Development With Hybrid Tables](https://www.youtube.com/watch?v=kBdz2BFxp6U)
