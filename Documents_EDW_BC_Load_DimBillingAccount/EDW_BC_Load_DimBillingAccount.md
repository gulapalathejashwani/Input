# EDW_BC_Load_DimBillingAccount – SSIS Data Flow Architecture Documentation

## Overview
This document provides a comprehensive, execution-accurate description of the SSIS package **EDW_BC_Load_DimBillingAccount**. It details all source tables, joins, transformations, SSIS variables, and target behaviors as implemented in the package's Data Flow Task.

---

## Data Flow Architecture Diagram

```mermaid
graph TD;
  OLE_SRC_GuideWire([OLE DB Source: Complex SQL<br/>Sources: bc_account, bc_ParentAcct, bc_accountcontact, bc_accountcontactrole, bctl_accountrole, bc_contact, bc_address, bctl_state, bctl_accounttype, bctl_billinglevel, bctl_customerservicetier, bc_securityzone, bctl_delinquencystatus, bctl_accountsegment])
  CNT_Source_Count([Row Count: User::SourceCount])
  DRV_BatchID([Derived Column: BatchID = @[User::BatchID]])
  LKP_DimBillingAccount([Lookup: DimBillingAccount])
  CSPL_CheckBeanVersion([Conditional Split:<br/>BeanVersion == EDWBeanVersion])
  CNT_Unchange_Count([Row Count: User::UnChangeCount])
  CNT_Update_Count([Row Count: User::UpdateCount])
  CNT_Insert_Count([Row Count: User::InsertCount])
  Load_DimBillingAccount([OLE DB Destination: [dbo].[DimBillingAccount]])
  CMD_Update_DimBillingAccount([OLE DB Command: UPDATE DimBillingAccount ...])

  OLE_SRC_GuideWire --> CNT_Source_Count
  CNT_Source_Count --> DRV_BatchID
  DRV_BatchID --> LKP_DimBillingAccount
  LKP_DimBillingAccount -->|Match| CSPL_CheckBeanVersion
  LKP_DimBillingAccount -->|No Match| CNT_Insert_Count
  CSPL_CheckBeanVersion -->|SameBeanVersion| CNT_Unchange_Count
  CSPL_CheckBeanVersion -->|DifferentBeanVersion| CNT_Update_Count
  CNT_Insert_Count --> Load_DimBillingAccount
  CNT_Update_Count --> CMD_Update_DimBillingAccount
```

---

## 1. Source Tables

### Main Data Extraction (OLE DB Source: OLE_SRC - GuideWire)
- **bc_account**
- **bc_ParentAcct**
- **bc_accountcontact**
- **bc_accountcontactrole**
- **bctl_accountrole**
- **bc_contact**
- **bc_address**
- **bctl_state**
- **bctl_accounttype**
- **bctl_billinglevel**
- **bctl_customerservicetier**
- **bc_securityzone**
- **bctl_delinquencystatus**
- **bctl_accountsegment**

### Lookup Reference Table
- **DimBillingAccount** (as reference in Lookup transformation)

---

## 2. Joins

### a. In OLE DB Source SQL
- Multiple joins across all above tables (see SSIS SQL for explicit join logic):
  - `LEFT JOIN`, `JOIN`, etc., as per source SQL
- **Join conditions and types are preserved as written in the OLE DB Source SQL.**

### b. Lookup Join
- **Left Table:** Data flow from OLE_SRC - GuideWire (column: PublicID)
- **Right Table:** DimBillingAccount (column: Publicid)
- **Join Type:** Lookup (equivalent to LEFT JOIN semantics for match/no match)
- **Condition:** `OLE_SRC.PublicID = DimBillingAccount.Publicid`

---

## 3. Transformations (Execution Order)

1. **Row Count – CNT - Source Count**
   - **Type:** Row Count
   - **Purpose:** Counts all source rows; result stored in SSIS variable `User::SourceCount`.

2. **Derived Column – DRV - BatchID & Legacy**
   - **Type:** Derived Column
   - **Description:** Adds or overwrites column `BatchID` with SSIS variable.
   - **Logic:** `BatchID = @[User::BatchID]` (SSIS runtime variable, not SQL)

3. **Lookup – LKP - DimBillingAccount**
   - **Type:** Lookup
   - **Description:** Looks up `BeanVersion` from DimBillingAccount based on `PublicID`.
   - **Logic:**
     - Reference SQL: `SELECT Publicid, BeanVersion FROM DimBillingAccount`
     - Join: `PublicID = Publicid`
     - Adds output column `EDWBeanVersion` to flow.

4. **Conditional Split – CSPL - Check BeanVersion**
   - **Type:** Conditional Split
   - **Description:** Routes rows by comparing `BeanVersion` (source) and `EDWBeanVersion` (lookup).
   - **Logic:**
     - Output 1 (SameBeanVersion): `BeanVersion == EDWBeanVersion`
     - Output 2 (DifferentBeanVersion): All others

5. **Row Count – CNT - Unchange Count**
   - **Type:** Row Count
   - **Purpose:** Counts rows where `BeanVersion` unchanged; stores in `User::UnChangeCount`.

6. **Row Count – CNT - Update Count**
   - **Type:** Row Count
   - **Purpose:** Counts rows where `BeanVersion` differs; stores in `User::UpdateCount`.

7. **Row Count – CNT - Insert Count**
   - **Type:** Row Count
   - **Purpose:** Counts new rows (no match in lookup); stores in `User::InsertCount`.

---

## 4. Target Table & Load Behavior

### a. OLE DB Destination: Load DimBillingAccount
- **Target Table:** `[dbo].[DimBillingAccount]` (Snowflake, UPPERCASE)
- **Load Type:** INSERT (for new rows)
- **Source:** Rows from CNT - Insert Count (lookup no match path)

### b. OLE DB Command: CMD - Update DimBillingAccount
- **Target Table:** `[dbo].[DimBillingAccount]` (Snowflake, UPPERCASE)
- **Load Type:** UPDATE (for changed rows)
- **Source:** Rows from CNT - Update Count (conditional split different path)
- **Update SQL:** (parameters mapped to columns)
  ```
UPDATE DimBillingAccount
SET DateUpdated = GetDate(),
    AccountCloseDate = ?,
    AccountCreationDate = ?,
    AccountName = ?,
    AccountNumber = ?,
    AccountTypeName = ?,
    AddressLine1 = ?,
    AddressLine2 = ?,
    AddressLine3 = ?,
    BatchId = ?,
    BeanVersion = ?,
    BillingLevelName = ?,
    City = ?,
    DeliquencyStatusName = ?,
    FirstName = ?,
    GWRowNumber = ?,
    IsActive = ?,
    LastName = ?,
    ParentAccountNumber = ?,
    PostalCode = ?,
    SecurityZone = ?,
    Segment = ?,
    ServiceTierName = ?,
    State = ?
WHERE PublicID = ?
  AND LegacySourceSystem = ?
  ```

---

## 5. SSIS Variables Used
- `@[User::BatchID]` – Used in Derived Column for `BatchID` and in parameter mapping for source SQL (incremental window)
- `@[User::IncrementDays]` – Used as parameter for source SQL date filtering
- `@[User::InsertCount]`, `@[User::UpdateCount]`, `@[User::UnChangeCount]`, `@[User::SourceCount]` – Row count variables for audit

**All SSIS variables are preserved verbatim as runtime variables.**

---

## 6. Execution Flow Summary
1. Extracts and joins data from multiple GuideWire and reference tables via complex SQL (see OLE DB Source)
2. Counts all rows (CNT - Source Count)
3. Adds `BatchID` using SSIS variable (DRV - BatchID & Legacy)
4. Looks up `BeanVersion` from DimBillingAccount (LKP)
5. Splits rows by comparing source and lookup `BeanVersion` (CSPL)
6. Routes:
   - **SameBeanVersion:** Counted as unchanged
   - **DifferentBeanVersion:** Counted as updates, routed to UPDATE command
   - **No Match:** Counted as inserts, routed to INSERT
7. Loads new rows via OLE DB Destination, updates via OLE DB Command
8. All row counts tracked for audit using SSIS variables

---

## 7. Full Source SQL (OLE_SRC - GuideWire)
```
WITH ParentAcct AS
(
    SELECT pa.OwnerID, CAST(act.AccountNumber as int) as ParentAccountNumber, CONCAT(pa.BeanVersion,act.BeanVersion) as BeanVersion, act.UpdateTime
    FROM bc_ParentAcct pa
    JOIN bc_account act ON act.ID = pa.ForeignEntityID
),
InsuredInfo AS
(
    SELECT ac.InsuredAccountID as AccountID, c.FirstName, c.LastName, a.AddressLine1, a.AddressLine2, a.AddressLine3, a.City, a.PostalCode, tls.NAME as State,
           CONCAT(ac.BeanVersion,acr.BeanVersion,c.BeanVersion ,a.BeanVersion) as BeanVersion, ac.UpdateTime as ac_UpdateTime, acr.UpdateTime as acr_UpdateTime,
           c.UpdateTime as c_UpdateTime, a.UpdateTime as a_UpdateTime
    FROM bc_accountcontact ac
    JOIN bc_accountcontactrole acr ON acr.AccountContactID = ac.ID
    JOIN bctl_accountrole tlar ON tlar.ID = acr.Role
    LEFT JOIN bc_contact c ON c.ID = ac.ContactID
    LEFT JOIN bc_address a ON a.ID = c.PrimaryAddressID
    LEFT JOIN bctl_state tls ON tls.ID = a.State
    WHERE tlar.TYPECODE = 'insured'
)
SELECT Distinct dt.AccountNumber, dt.AccountName, CAST(at.NAME  AS VARCHAR(50)) as AccountTypeName, ParentAcct.ParentAccountNumber,
    CAST(bl.NAME  AS VARCHAR(100)) as BillingLevelName, CAST(bas.NAME AS VARCHAR(50)) as Segment, CAST(cst.NAME AS VARCHAR(50)) as ServiceTierName,
    sz.Name as SecurityZone, InsuredInfo.FirstName, InsuredInfo.LastName, InsuredInfo.AddressLine1, InsuredInfo.AddressLine2, InsuredInfo.AddressLine3,
    CAST(InsuredInfo.City AS VARCHAR(50)) as City, CAST(InsuredInfo.State AS VARCHAR(50)) as State, CAST(InsuredInfo.PostalCode AS VARCHAR(50)) as PostalCode,
    dt.CloseDate as AccountCloseDate, dt.CreateTime as AccountCreationDate, CAST(tlds.NAME AS VARCHAR(50)) as DeliquencyStatusName,
    dt.FirstTwicePerMthInvoiceDOM as FirstTwicePerMonthInvoiceDayOfMonth, dt.SecondTwicePerMthInvoiceDOM as SecondTwicePerMonthInvoiceDayOfMonth,
    dt.PublicID, dt.ID as GWRowNumber, CAST(CONCAT(dt.BeanVersion ,ParentAcct.BeanVersion, sz.BeanVersion) AS VARCHAR(20)) as BeanVersion,
    CASE dt.Retired WHEN 0 THEN 1 ELSE 0 END as IsActive, 'WC' as LegacySourceSystem
FROM bc_account dt
LEFT JOIN bctl_accounttype at ON at.ID = dt.AccountType
LEFT JOIN ParentAcct ON ParentAcct.OwnerID = dt.ID
LEFT JOIN bctl_billinglevel bl ON bl.ID = dt.BillingLevel
LEFT JOIN bctl_customerservicetier cst ON cst.ID = dt.ServiceTier
LEFT JOIN bc_securityzone sz ON sz.ID = dt.SecurityZoneID
LEFT JOIN InsuredInfo ON InsuredInfo.AccountID = dt.ID
LEFT JOIN bctl_delinquencystatus tlds ON tlds.ID = dt.DelinquencyStatus
LEFT JOIN bctl_accountsegment bas ON bas.ID = dt.Segment
WHERE
(
  dt.UpdateTime >= DATEADD(D,?,CAST(GETDATE() AS DATE)) OR 
  ParentAcct.UpdateTime >= DATEADD(D,?,CAST(GETDATE() AS DATE)) OR
  sz.UpdateTime >= DATEADD(D,?,CAST(GETDATE() AS DATE)) OR
  InsuredInfo.ac_UpdateTime >= DATEADD(D,?,CAST(GETDATE() AS DATE)) OR
  InsuredInfo.acr_UpdateTime >= DATEADD(D,?,CAST(GETDATE() AS DATE)) OR
  InsuredInfo.c_UpdateTime >= DATEADD(D,?,CAST(GETDATE() AS DATE)) OR
  InsuredInfo.a_UpdateTime >= DATEADD(D,?,CAST(GETDATE() AS DATE))
)
```

---

## 8. Notes
- **All source and target tables are assumed to exist in Snowflake.**
- **No credentials or connection strings are stored in this documentation.**
- **SSIS runtime variables are preserved exactly as implemented.**