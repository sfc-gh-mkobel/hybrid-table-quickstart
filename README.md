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

In this HOL we will use Tasty Bytes order synthetic data to simulate a data serving use case. Each record in these datasets represents the state of a truck order.


### What You’ll Learn
- The basics of hybrid tables
- How to create and use hybrid tables
- The advantages of hybrid tables over standard tables
- Hybrid table unique characteristics like Indexes, primary keys, unique and foreign keys


### What You’ll Need

To complete this HOL, attendees need the following:
- Snowflake account with Hybrid Tables Enabled
- Account Admin and password which you should use to execute the HOL

**IMPORTANT**: <br>
At TECHUP, you will be provided with an account, a user name, and a password provided by Hol Helpers so no action is necessary on your end.
 
### Prerequisites
- Familiarity with the Snowflake Snowsight interface
- Basic experience using git
- Tasty Bytes V2 Installed on your account. If you don't have it install it by running [V2 Tasty Bytes Foundational Data Model - Setup](https://github.com/snowflakecorp/frostbytes/tree/main/Tasty%20Bytes/20%20-%20setup/V2)



## Lab 0: Lab Set Up

Duration: 5 Minutes

In this part of the lab, we will set up our Snowflake account, create new worksheets, role, database structures and create a Virtual Warehouse that we will use in this lab.

### Step 0.1 Creating a Worksheet

Within Worksheets, click the "+" button in the top-right corner of Snowsight and choose "SQL Worksheet"

![Screenshot 2023-10-03 at 13 23 58](https://github.com/snowflakecorp/techup-fy24snowday-hybrid_tables/assets/132544324/f0b2535a-30ff-4b7a-a1ef-1bf2b0b580e9)

Rename the Worksheet by clicking on the auto-generated Timestamp name and inputting "Hybrid Table - QuickStart"

### Step 0.2 Set Up

#### Create lab objects

Next we will create HYBRID_QUICKSTART_ROLE role, HYBRID_QUICKSTART_WH warehouse and HYBRID_QUICKSTART_DB database.

```sql
USE ROLE ACCOUNTADMIN;

-- Create role HYBRID_QUICKSTART_ROLE
CREATE OR REPLACE ROLE HYBRID_QUICKSTART_ROLE;
GRANT ROLE HYBRID_QUICKSTART_ROLE TO ROLE ACCOUNTADMIN;
-- Create HYBRID_QUICKSTART_WH warehouse
CREATE WAREHOUSE HYBRID_QUICKSTART_WH WAREHOUSE_SIZE = XSMALL, AUTO_SUSPEND = 300, AUTO_RESUME= TRUE;
GRANT OWNERSHIP ON WAREHOUSE HYBRID_QUICKSTART_WH TO ROLE HYBRID_QUICKSTART_ROLE;
GRANT CREATE DATABASE ON ACCOUNT TO ROLE HYBRID_QUICKSTART_ROLE;

-- Use role and create HYBRID_QUICKSTART_DB database and schema.
USE ROLE HYBRID_QUICKSTART_ROLE;
CREATE DATABASE HYBRID_QUICKSTART_DB;
CREATE SCHEMA DATA;
```
#### Create Hybrid Table and Bulk Load Data

You may bulk load data into hybrid tables by copying from a data stage or other tables (that is, using CTAS, COPY, or INSERT INTO … SELECT). Other methods of loading data into Snowflake tables (for example, Snowpipe) are not currently supported.

It is strongly recommended to bulk load data into a hybrid table using a CREATE TABLE … AS SELECT statement, as there are several optimizations which can only be applied to a data load as part of table creation. You need to define all keys, indexes, and constraints at the creation of a hybrid table. 

This DDL will create the structure for the ORDER_HEADER hybrid table
Note the primary key constraint on ORDER_ID column.

```sql
-- Set lab context use HYBRID_DB_USER_(USER_NUMBER) database and DATA schema
USE DATABASE HYBRID_QUICKSTART_DB;
USE SCHEMA DATA;

create or replace HYBRID TABLE ORDER_HEADER (
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
	primary key (ORDER_ID) rely ,
	foreign key (TRUCK_ID) references FROSTBYTE_TASTY_BYTES_UNISTORE.RAW_POS.TRUCK(TRUCK_ID) rely ,
	foreign key (LOCATION_ID) references FROSTBYTE_TASTY_BYTES_UNISTORE.RAW_POS.LOCATION(LOCATION_ID) rely ,
	foreign key (CUSTOMER_ID) references FROSTBYTE_TASTY_BYTES_UNISTORE.RAW_CUSTOMER.CUSTOMER_LOYALTY(CUSTOMER_ID) rely ,
	index IDX01_ORDER_TS(ORDER_TS),
	index IDX02_ORDER_STATUS(ORDER_STATUS),
	index IDX02_SHIFT(SHIFT)
)
AS
  (SELECT
     ORDER_ID,
     TRUCK_ID,
     LOCATION_ID,
     CUSTOMER_ID,
     SHIFT_START_TIME,
     SHIFT_END_TIME),
     ORDER_CHANNEL,
     ORDER_TS,
     SERVED_TS,
     ORDER_CURRENCY,
     ORDER_AMOUNT,
     ORDER_TAX_AMOUNT,
     ORDER_DISCOUNT_AMOUNT,
     ORDER_TOTAL,
     ORDER_STATUS
     from FROSTBYTE_TASTY_BYTES_UNISTORE.RAW_POS.ORDER_HEADER);

-- Not sure I need to grant any OWNERSHIP
-- grant OWNERSHIP to role HYBRID_ADMIN_ROLE
-- GRANT OWNERSHIP ON TABLE ORDER_STATE_STANDARD TO ROLE HYBRID_ADMIN_ROLE;
-- GRANT OWNERSHIP ON TABLE TRUCK_STANDARD TO ROLE HYBRID_ADMIN_ROLE;
-- GRANT OWNERSHIP ON SCHEMA DATA TO ROLE HYBRID_ADMIN_ROLE;
-- GRANT OWNERSHIP ON DATABASE HYBRID_DB TO ROLE HYBRID_ADMIN_ROLE;
```



















## Lab 1: Explore Data
Duration: 5 Minutes

In the pre-setup account step [Setting Up Hybrid](https://github.com/snowflakecorp/techup-fy24snowday-hybrid_tables/blob/main/Setting%20Up%20Hybrid.md) we created HYBRID_DB database, HYBRID_QUERY_WH warehouse, and schema DATA lets use them.

```sql
-- Lab 1
-- Set lab context
USE DATABASE HYBRID_DB;
USE SCHEMA DATA;
USE WAREHOUSE IDENTIFIER($USER_WAREHOUSE);
```
We also loaded data into the ORDER_STATE_HYBRID, ORDER_STATE_STANDARD, and ORDER_STATE_HYBRID_SECONDARY_INDEX tables. Now we can run a few queries and review some information to get familiar with it.

View tables properties and metadata. Note the value of the is_hybrid and rows column.
It's important to note that each table created in the set up step, tables ORDER_STATE_STANDARD, ORDER_STATE_HYBRID and ORDER_STATE_HYBRID_SECONDARY_INDEX contains the same number of records.

```sql
SHOW TABLES LIKE '%ORDER_STATE%';
```
Display information about the columns in the table. Note the primary key column and also that all three tables have table structure.
```sql
--Describe the columns in the table ORDER_STATE_STANDARD
DESC TABLE ORDER_STATE_STANDARD;
--Describe the columns in the table ORDER_STATE_HYBRID
DESC TABLE ORDER_STATE_HYBRID;
--Describe the columns in the table ORDER_STATE_HYBRID_SECONDARY_INDEX
DESC TABLE ORDER_STATE_HYBRID_SECONDARY_INDEX;
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
-- Simple query to look at 10 rows of data table ORDER_STATE_STANDARD
select * from ORDER_STATE_STANDARD limit 10;
-- Simple query to look at 10 rows of data from table ORDER_STATE_HYBRID
select * from ORDER_STATE_HYBRID limit 10;
-- Simple query to look at 10 rows of data table ORDER_STATE_HYBRID_SECONDARY_INDEX
select * from ORDER_STATE_HYBRID_SECONDARY_INDEX limit 10;
```

<!-- ------------------------ -->
## Lab 2: Latency Test Using Streamlit App
Duration: 15 Minutes

In this part of the lab, we will set up a [streamlit](https://docs.snowflake.com/en/developer-guide/streamlit/about-streamlit) App that we will use for running queries and comparing the latency of the two types of tables standard vs. hybrid.
To simulate a single-row lookup, all the queries used in this test will include a WHERE clause with a filter on the ORDER_ID column.
It is important to note that this test is not designed to simulate load or concurrency, but rather runs the queries sequentially and its primary purpose is to provide us with a clearer and more detailed visualization of the latency associated with each table type.

### Step 2.1 Streamlit App Set Up

First, we will create streamlit schema and grant create streamlit privilege to HYBRID_ADMIN_ROLE role
```sql
-- Lab 2
-- Set lab context use HYBRID_DB_USER_(USER_NUMBER) database
USE DATABASE IDENTIFIER($USER_DATABASE);
CREATE SCHEMA STREAMLIT_APP;
GRANT CREATE STREAMLIT ON SCHEMA STREAMLIT_APP TO ROLE IDENTIFIER($USER_ROLE);
```

Now we will create the Streamlit app by following these steps:

1. Click on Snowsight Home Button
2. On the upper left side of the page, click the user name and make sure HYBRID_HOL_USER_(USER_NUMBER)_ROLE role is selected


![Screenshot 2023-10-18 at 9 20 17](https://github.com/snowflakecorp/techup-fy24snowday-hybrid_tables/assets/132544324/88788813-5f0f-446e-808f-742abb2eb9f9)


3. In the left navigation bar, select Streamlit.
4. Click the blue "+ Streamlit App" button located on the right upper corner
   
![Screenshot 2023-10-18 at 9 22 00](https://github.com/snowflakecorp/techup-fy24snowday-hybrid_tables/assets/132544324/488bd26b-3eac-4dba-b60c-4a17574f22e7)


5. In the "Create Streamlit App" popup enter the following:
  * "Latency Test App" as App name
  * Select user warehouse from the list of available warehouses
  * Select user database from the list of available databases
  * Select "STREAMLIT_APP" from the list of available schemas
  * Click create button

![Screenshot 2023-10-18 at 9 22 42](https://github.com/snowflakecorp/techup-fy24snowday-hybrid_tables/assets/132544324/a061b5dd-54cf-4960-b4af-065164e5c030)



6. Replace the default example code with the new Hybrid Latency Test App code located in the git repository file [streamlit.py](https://github.com/snowflakecorp/techup-fy24snowday-hybrid_tables/blob/main/streamlit.py)

8. Click the blue "Run" button on the right upper corner of the page

![Screenshot 2023-10-05 at 15 19 39](https://github.com/snowflakecorp/techup-fy24snowday-hybrid_tables/assets/132544324/0c56a5df-3f2e-44e3-bac5-9f9d7a0537ec)

Now we should see the following Streamlit App:

![Screenshot 2023-10-18 at 9 25 05](https://github.com/snowflakecorp/techup-fy24snowday-hybrid_tables/assets/132544324/b902b97b-b28d-4740-a1ba-577d6731a08b)



### Step 2.2 Run Test to Compare Two Tables

The streamlit app provides 2 query types to select:

1. Select 
2. Update 

First, select a query type to run and then set the "Number of queries to run" using the slide object.
Click the submit button to start running the use case.

Once submitted the app will run the following logic:
1. Query the standard table and select randomly X number of order_id where X is the value of "Number of queries to run" slide object.
2. For each order_id, execute the same type of query on both the hybrid and standard table and save latency results.
3. Present the results for further examination and comparison.


Use the streamlit app to assess and compare each use case.
For each use case compare the latency results and analyze the differences in performance between the two types of tables.

Additionally, open the [QUERY HISTORY](https://docs.snowflake.com/en/user-guide/ui-snowsight-activity#label-snowsight-activity-query-history) and review and compare the execution details for each type of query.


<!-- ------------------------ -->
## Lab 3: Secondary Index Constraint Performance

Duration: 5 Minutes

In this lab, we will compare two tables: "ORDER_STATE_HYBRID_SECONDARY_INDEX," where the TRUCK_ID column is configured as a secondary index, and "ORDER_STATE_HYBRID," which lacks a secondary index. 
We anticipate that the latency results for running the query, which includes a filter on TRUCK_ID column, on the former table will be significantly faster compared to running the same query on the ORDER_STATE_HYBRID table, as it does not have the TRUCK_ID column configured as a secondary index.

### Step 3.1 Compare Two Tables 

First let's get a TRUCK_ID value that we will use in the next queries

```sql
-- Lab 3
-- Set lab context
USE DATABASE HYBRID_DB;
USE SCHEMA DATA;
set TRUCK_ID = (SELECT TRUCK_ID FROM HYBRID_DB.DATA.ORDER_STATE_HYBRID LIMIT 1);
--TRUCK_ID variable value
SELECT $TRUCK_ID AS TRUCK_ID_VALUE;

```

To conduct the latency comparison between these two tables, we will execute two queries and compare their respective latencies.

```sql
-- select from table HYBRID_DB.DATA.ORDER_STATE_HYBRID_SECONDARY_INDEX
SELECT * FROM ORDER_STATE_HYBRID_SECONDARY_INDEX WHERE TRUCK_ID  = $TRUCK_ID;

-- select from table HYBRID_DB.DATA.ORDER_STATE_HYBRID
SELECT * FROM ORDER_STATE_HYBRID WHERE TRUCK_ID  = $TRUCK_ID;
```

Open the [QUERY HISTORY](https://docs.snowflake.com/en/user-guide/ui-snowsight-activity#label-snowsight-activity-query-history) and review and compare the execution details for each type of query.

For each query:
1. Click on Snowsight Home Button
2. In the left navigation bar, select Activity->Query History.
3. Search and click the query
4. Click the "Query Profile" tab
5. In the Profile Overview review the "Total Execution Time"
6. Click on the first "TableScan" node in the tree.
7. Review the node attributes metadata on the bottom right side of the page and locate the Scan Mode attribute

Please note that for the first query, as we are querying a table with a secondary index configured, the scan mode is "row-based with index" while for the second query, the scan mode is columnar.
It's important to highlight that 'row-based with index' is expected to be much faster compared to a columnar scan.

<!-- ------------------------ -->
## Lab 4: Unique and Foreign Keys Constraints

Duration: 10 Minutes

In this lab, we will create two hybrid tables: TRUCK_HYBRID and ORDER_STATE_HYBRID_CONSTRAINTS that we will use for testing unique and foreign key constraints.
The table TRUCK_HYBRID contains additional data on trucks like truck model or truck opening date.
The table ORDER_STATE_HYBRID_CONSTRAINTS contains data on orders.
In table ORDER_STATE_HYBRID_CONSTRAINTS we will configure TRUCK_ID as a FOREIGN KEY reference to column TRUCK_ID in table TRUCK_HYBRID and also configure FRANCHISEE_EMAIL as a unique column.

### Step 4.1 Create Tables

This DDL will create the structure for the TRUCK_HYBRID hybrid table
Note the primary key constraint on TRUCK_ID column.

```sql
-- Lab 4
-- Set lab context use HYBRID_DB_USER_(USER_NUMBER) database and DATA schema
USE DATABASE IDENTIFIER($USER_DATABASE);
USE SCHEMA DATA;


CREATE HYBRID TABLE TRUCK_HYBRID (
	TRUCK_ID NUMBER PRIMARY KEY,
	MENU_TYPE_ID NUMBER,
	PRIMARY_CITY VARCHAR,
	REGION VARCHAR,
	ISO_REGION VARCHAR,
	COUNTRY VARCHAR,
	ISO_COUNTRY_CODE VARCHAR,
	FRANCHISE_FLAG NUMBER,
	YEAR NUMBER,
	MAKE VARCHAR,
	MODEL VARCHAR,
	EV_FLAG NUMBER,
	FRANCHISE_ID NUMBER,
	TRUCK_OPENING_DATE DATE
);
```

This DDL will create the structure for the ORDER_STATE_HYBRID_CONSTRAINTS hybrid table.
Note the foreign key constraint on TRUCK_ID column and the unique constraint on FRANCHISEE_EMAIL column.

```sql
CREATE HYBRID TABLE ORDER_STATE_HYBRID_CONSTRAINTS (
       ORDER_ID NUMBER NOT NULL PRIMARY KEY,
       TRUCK_ID NUMBER NOT NULL FOREIGN KEY REFERENCES TRUCK_HYBRID (TRUCK_ID),
       TRUCK_BRAND_NAME VARCHAR  NOT NULL,
       FRANCHISE_ID   NUMBER NOT NULL,
       FRANCHISEE_FIRST_NAME VARCHAR  NOT NULL,
       FRANCHISEE_LAST_NAME VARCHAR   NOT NULL,
       FRANCHISEE_EMAIL VARCHAR   NOT NULL UNIQUE,
       PRIMARY_CITY VARCHAR  NOT NULL,
       REGION VARCHAR  NOT NULL,
       COUNTRY VARCHAR NOT NULL,
       DETAIL_COUNT NUMBER NOT NULL,
       TOTAL_SALES NUMBER NOT NULL,
       CUSTOMER_SENTIMENT_SCORE FLOAT NOT NULL,
       FOOD_TASTE_RATE FLOAT NOT NULL,
       FOOD_PORTION_RATE FLOAT NOT NULL,
       FOOD_FRESH_AND_HOT FLOAT NOT NULL,
       SERVICE_EASY_TO_ORDER_RATE FLOAT NOT NULL,
       SERVICE_FASTNESS_RATE FLOAT NOT NULL,
       PRICE_ACCEPTABILITY FLOAT NOT NULL,
       PRICE_TO_FOOD_RATIO FLOAT NOT NULL,
       LOCATION_CONVIENENTLY FLOAT NOT NULL,
       LOCATION_EASILY_ACCESSIBLE FLOAT NOT NULL,
       EXPERIENCE_OVERALL FLOAT NOT NULL,
       EXPERIENCE_RECOMMEND FLOAT NOT NULL,
       ORDER_COMMENT VARCHAR
);
```

Display information about the columns in the table. Note the primary key and unique key columns.
```sql
--Describe the columns in the table ORDER_STATE_HYBRID_CONSTRAINTS
DESC TABLE ORDER_STATE_HYBRID_CONSTRAINTS;
```

### Step 4.2 Test Unique and Foreign Keys Constraints

#### Insert Foreign Keys Constraints

In this step we will test foreign Keys constraint.
First, we will try to insert a new record to table ORDER_STATE_HYBRID_CONSTRAINTS. 
It is expected that the insert statement would fail since TRUCK_HYBRID table does not contain a record with truck_id 377, which is the foreign key reference.

```sql
insert into ORDER_STATE_HYBRID_CONSTRAINTS values (1,377,'Smoky BBQ',276,'Christopher','Sutton','jdoe@gmail.com','Stockholm','Stockholm  län','Sweden',1,42,0.9669457292,1,1,4,1,1,3,3,5,2,1,3,'');
```
The statement should fail and we should receive the following error message:

"Foreign key constraint "SYS_INDEX_ORDER_STATE_HYBRID_CONSTRAINTS_FOREIGN_KEY_TRUCK_ID_TRUCK_HYBRID_TRUCK_ID" is violated."

Now we will first insert the record to table TRUCK_HYBRID and only then insert a new record to table ORDER_STATE_HYBRID_CONSTRAINTS:

```sql
insert into TRUCK_HYBRID values (377,2,'Stockholm','Stockholm län','Stockholm','Sweden','SE',1,2001,'Freightliner','MT45 Utilimaster',0,276,'2020-10-01');
insert into ORDER_STATE_HYBRID_CONSTRAINTS values (1,377,'Smoky BBQ',276,'Christopher','Sutton','jdoe@gmail.com','Stockholm','Stockholm  län','Sweden',1,42,0.9669457292,1,1,4,1,1,3,3,5,2,1,3,'');
```

Both statements should run successfully.


#### Unique Constraints

In this step, we will test Unique Constraint which ensures that all values in a column are different.
In the previous step, we already inserted a record into table ORDER_STATE_HYBRID_CONSTRAINTS where the email column value is 'jdoe@gmail.com'.
To view the record run the following statement:

```sql
select * from ORDER_STATE_HYBRID_CONSTRAINTS where ORDER_ID = 1 and FRANCHISEE_EMAIL = 'jdoe@gmail.com';
```

Due to the unique constraint, if we attempt to insert two records with the same email address, the statement will fail.
To test it run the following statement:

```sql
insert into ORDER_STATE_HYBRID_CONSTRAINTS values (2,377,'Smoky BBQ',276,'Christopher','Sutton','jdoe@gmail.com','Stockholm','Stockholm  län','Sweden',1,42,0.9669457292,1,1,4,1,1,3,3,5,2,1,3,'');
```

Since we configured the column FRANCHISEE_EMAIL in table ORDER_STATE_HYBRID_CONSTRAINTS as UNIQUE the statement failed and we should receive the following error message:

"Duplicate key value violates unique constraint "SYS_INDEX_ORDER_STATE_HYBRID_CONSTRAINTS_UNIQUE_FRANCHISEE_EMAIL"

#### Truncated Active Foreign Key Constraint

In this step, we will test that the table referenced by a foreign key constraint cannot be truncated as long as the foreign key relationship exists.
To test it run the following statement:

```sql
TRUNCATE TABLE TRUCK_HYBRID;
```

The statement should fail and we should receive the following error message:

"Hybrid table 'TRUCK_HYBRID' cannot be truncated as it is involved in active foreign key constraints."


#### Delete Foreign Key Constraint

In this step, we will test that a record referenced by a foreign key constraint cannot be deleted as long as the foreign key reference relationship exists.

To test it run the following statement:

```sql
DELETE FROM TRUCK_HYBRID WHERE TRUCK_ID = 377;
```

The statement should fail and we should receive the following error message:
"Foreign keys that reference key values still exist."

In order to be able to delete a record referenced by a foreign key constraint you need first to delete the reference record in table ORDER_STATE_HYBRID_CONSTRAINTS and only then delete the referenced by record in table TRUCK_HYBRID
To test it run the following statement:

```sql
DELETE FROM ORDER_STATE_HYBRID_CONSTRAINTS WHERE ORDER_ID = 1;
DELETE FROM TRUCK_HYBRID WHERE TRUCK_ID = 377;
```

Both statements should run successfully.

<!-- ------------------------ -->
## Lab 5: Join Hybrid Table and Standard Snowflake Table
Duration: 5 Minutes

In this part of the lab, we will test the join between hybrid and standard tables. We will create a new TRUCK_STANDARD standard table and we will use it to join with table ORDER_STATE_HYBRID.

### Step 5.1 Explore Data 

In the Setting Up Hybrid step, we already loaded data into the TRUCK_STANDARD tables. Now we can run a few queries and review some information to get familiar with it.
```sql

-- Lab 5
-- Set lab context
USE DATABASE HYBRID_DB;
USE SCHEMA DATA;

-- Simple query to look at 10 rows of data from table TRUCK_STANDARD
select * from TRUCK_STANDARD limit 10;
-- Simple query to look at 10 rows of data from table ORDER_STATE_HYBRID
select * from ORDER_STATE_HYBRID limit 10;
```
### Step 5.2 Join Hybrid Table and Standard Snowflake Table

In order to test the join of the hybrid table ORDER_STATE_HYBRID with the standard table TRUCK_STANDARD, let's run the join statement.

```sql
-- Set ORDER_ID variable
set ORDER_ID = (select order_id from ORDER_STATE_HYBRID limit 1);

-- Join tables ORDER_STATE_HYBRID and TRUCK_STANDARD
select HY.*,ST.* from ORDER_STATE_HYBRID as HY join TRUCK_STANDARD as ST on HY.truck_id = ST.TRUCK_ID where HY.ORDER_ID = $ORDER_ID;
```

After executing the join statement examine and analyze the data in the result set.

<!-- ------------------------ -->
## Lab 6: Security / Governance
Duration: 10 Minutes

In this lab, we will demonstrate that the security and governance functionalities that have been applied to the standard table are also exist for the hybrid table. 

### Step 6.1 Hybrid Table Access Control and User Management

Role-based access control (RBAC) in Snowflake for hybrid tables is the same as standard tables.
The purpose of this exercise is to give you a chance to see how you can manage access to hybrid table data in Snowflake by granting privileges to some roles.

First we will insert data to tables TRUCK_HYBRID and ORDER_STATE_HYBRID_CONSTRAINTS

```sql
-- Lab 6
-- Set lab context
-- Use HYBRID_DB_USER_(USER_NUMBER) database
USE DATABASE IDENTIFIER($USER_DATABASE);
USE SCHEMA DATA;

insert into TRUCK_HYBRID values (377,2,'Stockholm','Stockholm län','Stockholm','Sweden','SE',1,2001,'Freightliner','MT45 Utilimaster',0,276,'2020-10-01');
insert into ORDER_STATE_HYBRID_CONSTRAINTS values (1,377,'Smoky BBQ',276,'Christopher','Sutton','jdoe@gmail.com','Stockholm','Stockholm  län','Sweden',1,42,0.9669457292,1,1,4,1,1,3,3,5,2,1,3,'');
```

Then we will create a new HYBRID_HOL_BI_USER_(user_number)_ROLE role

```sql
USE ROLE ACCOUNTADMIN;
SET BI_USER_ROLE = 'HYBRID_HOL_BI_USER_'||$USER_NUMBER||'_ROLE';
CREATE ROLE IDENTIFIER($BI_USER_ROLE);
SET MY_USER = CURRENT_USER();
GRANT ROLE IDENTIFIER($BI_USER_ROLE) TO USER IDENTIFIER($MY_USER);
```

Then we will grant USAGE privileges to HYBRID_QUERY_USER_(USER_NUMBER)_WH warehouse, HYBRID_DB_USER_(USER_NUMBER) database, and all its schemas to role HYBRID_HOL_BI_USER_(user_number)_ROLE

```sql
-- Use HYBRID_HOL_USER_(USER_NUMBER)_ROLE role
USE ROLE IDENTIFIER($USER_ROLE);
GRANT USAGE ON WAREHOUSE IDENTIFIER($USER_WAREHOUSE) TO ROLE IDENTIFIER($BI_USER_ROLE);
GRANT USAGE ON DATABASE IDENTIFIER($USER_DATABASE) TO ROLE IDENTIFIER($BI_USER_ROLE);
GRANT USAGE ON ALL SCHEMAS IN DATABASE IDENTIFIER($USER_DATABASE) TO ROLE IDENTIFIER($BI_USER_ROLE);
```

Use the newly created role and try to select some data from ORDER_STATE_HYBRID hybrid table

```sql
-- Use HYBRID_HOL_BI_USER_(user_number)_ROLE role
USE ROLE IDENTIFIER($BI_USER_ROLE);
-- Use HYBRID_DB_USER_(USER_NUMBER) database
USE DATABASE IDENTIFIER($USER_DATABASE);
USE SCHEMA DATA;

select * from ORDER_STATE_HYBRID_CONSTRAINTS limit 10;
```
We’re not able to select any data. That’s because the role HYBRID_HOL_BI_USER_(user_number)_ROLE has not been granted the necessary privileges.

Use role HYBRID_HOL_USER_(USER_NUMBER)_ROLE and grant role HYBRID_HOL_BI_USER_(user_number)_ROLE select privileges on table ORDER_STATE_HYBRID_CONSTRAINTS.

```sql
-- Use HYBRID_HOL_USER_(USER_NUMBER)_ROLE role
USE ROLE IDENTIFIER($USER_ROLE);
GRANT SELECT ON TABLE ORDER_STATE_HYBRID_CONSTRAINTS TO ROLE IDENTIFIER($BI_USER_ROLE);
```

Try again to select some data from ORDER_STATE_HYBRID hybrid table

```sql
-- Use HYBRID_HOL_BI_USER_(user_number)_ROLE role
USE ROLE IDENTIFIER($BI_USER_ROLE);
select * from ORDER_STATE_HYBRID_CONSTRAINTS limit 10;
```

This time it worked! This is because HYBRID_BI_ROLE role has the appropriate privileges at all levels of the hierarchy.

### Step 6.2 Hybrid Table Masking Policy

In this step, we will create a new masking policy object and apply the masking policy to a column FRANCHISEE_EMAIL in a table ORDER_STATE_HYBRID_CONSTRAINTS using an ALTER TABLE … ALTER COLUMN statement.

First, we will create a new masking policy.

```sql
-- Use HYBRID_HOL_USER_(USER_NUMBER)_ROLE role
USE ROLE IDENTIFIER($USER_ROLE);
-- Use HYBRID_DB_USER_(USER_NUMBER) database
USE DATABASE IDENTIFIER($USER_DATABASE);
-- Use DATA schema
USE SCHEMA DATA;

--full column masking version, always masks
create masking policy hide_column_values as
(col_value varchar) returns varchar ->
  case
     WHEN current_role() IN ($USER_ROLE) THEN col_value
    else '***MASKED***'
  end;
```

Apply the policy to the table.

```sql
-- set masking policy
alter table ORDER_STATE_HYBRID_CONSTRAINTS modify column FRANCHISEE_EMAIL
set masking policy hide_column_values using (FRANCHISEE_EMAIL);
```

Since we are using the role HYBRID_HOL_USER_(USER_NUMBER)_ROLE column FRANCHISEE_EMAIL should not be masked.
Run the statement to see if it has the desired effect

```sql
select * from ORDER_STATE_HYBRID_CONSTRAINTS limit 10;
```

Since we are using role HYBRID_HOL_BI_USER_(user_number)_ROLE column FRANCHISEE_EMAIL should be masked.
Run the statement to see if it has the desired effect

```sql
-- Use HYBRID_HOL_BI_USER_(user_number)_ROLE role
USE ROLE IDENTIFIER($BI_USER_ROLE);
select * from ORDER_STATE_HYBRID_CONSTRAINTS;
```

<!-- ------------------------ -->
## Lab 7: Cleanup

To clean up your Snowflake environment you can run the following SQL Statements.

```sql
-- Lab 7
-- Set lab context
-- Use HYBRID_DB_USER_(USER_NUMBER) database and HYBRID_HOL_USER_(USER_NUMBER)_ROLE role
USE DATABASE IDENTIFIER($USER_DATABASE);
USE ROLE IDENTIFIER($USER_ROLE);
DROP STREAMLIT IF EXISTS "Latency Test App";
DROP DATABASE IDENTIFIER($USER_DATABASE);
DROP WAREHOUSE IDENTIFIER($USER_WAREHOUSE);
USE ROLE ACCOUNTADMIN;
DROP ROLE IDENTIFIER($BI_USER_ROLE);

The last step is to manually delete "Hybrid Table - HOL" workspace.




