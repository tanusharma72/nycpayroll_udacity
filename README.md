# NYC Payroll Data Engineering Project

This project demonstrates an end-to-end data engineering pipeline on Azure using NYC Payroll data. It covers data ingestion, transformation, aggregation, and analytics using Azure Data Factory, Azure SQL Database, Azure Data Lake Gen2, and Azure Synapse Analytics.

---

## Architecture Overview

**Azure Services Used**

* Azure Data Lake Storage Gen2
* Azure SQL Database
* Azure Data Factory (ADF)
* Azure Synapse Analytics

---

## Step 1: Prepare the Data Infrastructure

### 1.1 Create Azure Resources

Create the following Azure resources:

* Azure Data Lake Storage Gen2 (Storage Account)
* Azure SQL Database named **`db_nycpayroll`**
* Azure Data Factory
* Azure Synapse Analytics Workspace



### 1.2 Configure Azure Data Lake Storage Gen2

1. Create a container in the storage account.
2. Inside the container, create the following directories:

   * `dirpayrollfiles`
   * `dirhistoryfiles`
   * `dirstaging`



### 1.3 Upload Source Files

Upload the files as shown below:

**`dirpayrollfiles`**

* `EmpMaster.csv`
* `AgencyMaster.csv`
* `TitleMaster.csv`
* `nycpayroll_2021.csv`

**`dirhistoryfiles`**

* `nycpayroll_2020.csv`



## Step 2: Create External Objects in Synapse Analytics

Run the following commands in the Synapse SQL editor (Serverless SQL Pool):


CREATE DATABASE databasename;
USE databasename;


### 2.1 External Data Source


CREATE EXTERNAL DATA SOURCE NYC_ADLS
WITH (
    LOCATION = 'https://practiceaccount.dfs.core.windows.net/payrollfiles'
);


### 2.2 External File Format

 
CREATE EXTERNAL FILE FORMAT CSV_Format
WITH (
    FORMAT_TYPE = DELIMITEDTEXT,
    FORMAT_OPTIONS (
        FIELD_TERMINATOR = ',',
        FIRST_ROW = 2
    )
);


### 2.3 Create External View /table for Payroll Summary


CREATE VIEW dbo.NYC_Payroll_Summary
AS
SELECT
    CAST(FiscalYear AS INT) AS FiscalYear,
    AgencyName,
    CAST(TotalPaid AS FLOAT) AS TotalPaid
FROM OPENROWSET(
    BULK 'dirstaging/*.csv',
    DATA_SOURCE = 'NYC_ADLS',
    FORMAT = 'CSV'
)
WITH (
    FiscalYear  VARCHAR(10),
    AgencyName  VARCHAR(50),
    TotalPaid   VARCHAR(20)
) AS Payroll
WHERE FiscalYear <> 'FiscalYear';
GO
```

---

## Step 3: Create Tables in Azure SQL Database

Run the following scripts in Azure SQL Database:

### 3.1 Summary Table

```sql
CREATE TABLE dbo.NYC_Payroll_Summary (
    FiscalYear INT NULL,
    AgencyName VARCHAR(50) NULL,
    TotalPaid FLOAT NULL
);
GO
```

### 3.2 Payroll Data Tables (2020 & 2021)

```sql
CREATE TABLE dbo.NYC_Payroll_Data_2020 (
    FiscalYear INT NULL,
    PayrollNumber INT NULL,
    AgencyID VARCHAR(10) NULL,
    AgencyName VARCHAR(50) NULL,
    EmployeeID VARCHAR(10) NULL,
    LastName VARCHAR(20) NULL,
    FirstName VARCHAR(20) NULL,
    AgencyStartDate DATE NULL,
    WorkLocationBorough VARCHAR(50) NULL,
    TitleCode VARCHAR(10) NULL,
    TitleDescription VARCHAR(100) NULL,
    LeaveStatusasofJune30 VARCHAR(50) NULL,
    BaseSalary FLOAT NULL,
    PayBasis VARCHAR(50) NULL,
    RegularHours FLOAT NULL,
    RegularGrossPaid FLOAT NULL,
    OTHours FLOAT NULL,
    TotalOTPaid FLOAT NULL,
    TotalOtherPay FLOAT NULL
);
GO
```

```sql
CREATE TABLE dbo.NYC_Payroll_Data_2021 (
    FiscalYear INT NULL,
    PayrollNumber INT NULL,
    AgencyID VARCHAR(10) NULL,
    AgencyName VARCHAR(50) NULL,
    EmployeeID VARCHAR(10) NULL,
    LastName VARCHAR(20) NULL,
    FirstName VARCHAR(20) NULL,
    AgencyStartDate DATE NULL,
    WorkLocationBorough VARCHAR(50) NULL,
    TitleCode VARCHAR(10) NULL,
    TitleDescription VARCHAR(100) NULL,
    LeaveStatusasofJune30 VARCHAR(50) NULL,
    BaseSalary FLOAT NULL,
    PayBasis VARCHAR(50) NULL,
    RegularHours FLOAT NULL,
    RegularGrossPaid FLOAT NULL,
    OTHours FLOAT NULL,
    TotalOTPaid FLOAT NULL,
    TotalOtherPay FLOAT NULL
);
GO
```

### 3.3 Master Tables

```sql
CREATE TABLE dbo.EmpMaster (
    EmployeeID VARCHAR(10) NULL,
    LastName VARCHAR(20) NULL,
    FirstName VARCHAR(20) NULL
);
GO
```

```sql
CREATE TABLE dbo.TitleMaster (
    TitleCode VARCHAR(10) NULL,
    TitleDescription VARCHAR(100) NULL
);
GO
```

```sql
CREATE TABLE dbo.AgencyMaster (
    AgencyID VARCHAR(10) NULL,
    AgencyName VARCHAR(50) NULL
);
GO
```

---

## Step 4: Create Linked Services in Azure Data Factory

Create the following linked services:

* Azure Data Lake Storage Gen2
* Azure SQL Database

---

## Step 5: Create Datasets in Azure Data Factory

### 5.1 Data Lake Datasets

Create datasets for:

* 5 source CSV files
* 1 staging directory (`dirstaging`)

### 5.2 SQL Database Datasets

Create datasets for:

* 5 SQL tables
* 1 summary table

**Total datasets:** 12

---

## Step 6: Create Data Flows

* Create individual data flows to load all 5 CSV files from Data Lake into SQL DB
* Create a separate data flow for summary aggregation

---

## Step 7: Data Aggregation & Parameterization

### Summary Data Flow Logic

1. Create a new data flow named **`Dataflow_Summary`**
2. Add two sources:

   * Payroll 2020 data from SQL DB
   * Payroll 2021 data from SQL DB
3. Add **Select** transformations if schema mapping is required
4. Add a **Union** transformation to merge 2020 and 2021 data
5. Add a **Filter** transformation

**Parameter Configuration**

* Parameter Name: `dataflow_param_fiscalyear`
* Type: Integer
* Default Value: 2021

**Filter Expression**

```text
toInteger(FiscalYear) >= $dataflow_param_fiscalyear
```

6. Add a **Derived Column** transformation

   * Column Name: `TotalPaid`
   * Expression:

   ```text
   iifNull(toFloat(RegularGrossPaid), 0.0)
   + iifNull(toFloat(TotalOTPaid), 0.0)
   + iifNull(toFloat(TotalOtherPay), 0.0)
   ```

7. Add an **Aggregate** transformation

   * Group By: `AgencyName`, `FiscalYear`
   * Aggregation: `sum(TotalPaid)`

8. Add two **Sink** transformations:

   * Sink 1: SQL DB Summary Table (Enable *Truncate Table*)
   * Sink 2: Azure Synapse Analytics Summary Table (Enable *Truncate Table*)

---

## Step 8: Pipeline Creation & Execution

1. Create a pipeline in Azure Data Factory to:

   * Load source data into SQL DB
   * Execute summary aggregation data flow
   * Store results in SQL DB and Synapse Analytics
2. Trigger the pipeline manually or using a schedule
3. Monitor pipeline execution
4. Verify data in SQL DB and Synapse Analytics

---

## Final Verification

* Ensure all pipelines succeed
* Validate record counts and aggregation results
* Query Synapse external view for analytics

---

✅ **Project Completed Successfully**
