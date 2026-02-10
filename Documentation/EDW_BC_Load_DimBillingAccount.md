# EDW_BC_Load_DimBillingAccount

## Data Flow Overview

```
[OLE_SRC - GuideWire]
   |
   v
[CNT - Source Count]
   |
   v
[DRV - BatchID & Legacy]
   |
   v
[LKP - DimBillingAccount]
   |------------------------------|
   |                              |
   v                              v
[Lookup Match Output]        [Lookup No Match Output]
   |                              |
   v                              v
[CSPL - Check BeanVersion]   [CNT - Insert Count]
   |---------------------|
   |                     |
   v                     v
[SameBeanVersion]   [DifferentBeanVersion]
   |                     |
   v                     v
[CNT - Unchange Count] [CNT - Update Count]
   |                     |
   v                     v
                         [CMD - Update DimBillingAccount]

[Load DimBillingAccount]
```

## Source Tables

- **BC_ACCOUNT**
- **BC_PARENTACCT**
- **BC_ACCOUNTCONTACT**
- **BC_ACCOUNTCONTACTROLE**
- **BCTL_ACCOUNTROLE**
- **BC_CONTACT**
- **BC_ADDRESS**
- **BCTL_STATE**
- **BCTL_ACCOUNTTYPE**
- **BCTL_BILLINGLEVEL**
- **BCTL_CUSTOMERSERVICETIER**
- **BC_SECURITYZONE**
- **BCTL_DELINQUENCYSTATUS**
- **BCTL_ACCOUNTSEGMENT**
- **DIMBILLINGACCOUNT** (for lookup)

## Joins

| Left Table           | Right Table                  | Join Type | Condition                                      |
|----------------------|-----------------------------|-----------|------------------------------------------------|
| BC_ACCOUNT           | BCTL_ACCOUNTTYPE            | LEFT      | BCTL_ACCOUNTTYPE.ID = BC_ACCOUNT.ACCOUNTTYPE    |
| BC_ACCOUNT           | BC_PARENTACCT               | LEFT      | BC_PARENTACCT.OWNERID = BC_ACCOUNT.ID           |
| BC_PARENTACCT        | BC_ACCOUNT                  | INNER     | BC_ACCOUNT.ID = BC_PARENTACCT.FOREIGNENTITYID   |
| BC_ACCOUNT           | BCTL_BILLINGLEVEL           | LEFT      | BCTL_BILLINGLEVEL.ID = BC_ACCOUNT.BILLINGLEVEL  |
| BC_ACCOUNT           | BCTL_CUSTOMERSERVICETIER    | LEFT      | BCTL_CUSTOMERSERVICETIER.ID = BC_ACCOUNT.SERVICETIER |
| BC_ACCOUNT           | BC_SECURITYZONE             | LEFT      | BC_SECURITYZONE.ID = BC_ACCOUNT.SECURITYZONEID  |
| BC_ACCOUNT           | INSUREDINFO                 | LEFT      | INSUREDINFO.ACCOUNTID = BC_ACCOUNT.ID           |
| BC_ACCOUNT           | BCTL_DELINQUENCYSTATUS      | LEFT      | BCTL_DELINQUENCYSTATUS.ID = BC_ACCOUNT.DELINQUENCYSTATUS |
| BC_ACCOUNT           | BCTL_ACCOUNTSEGMENT         | LEFT      | BCTL_ACCOUNTSEGMENT.ID = BC_ACCOUNT.SEGMENT     |
| BC_ACCOUNTCONTACT    | BC_ACCOUNTCONTACTROLE       | INNER     | BC_ACCOUNTCONTACTROLE.ACCOUNTCONTACTID = BC_ACCOUNTCONTACT.ID |
| BC_ACCOUNTCONTACTROLE| BCTL_ACCOUNTROLE            | INNER     | BCTL_ACCOUNTROLE.ID = BC_ACCOUNTCONTACTROLE.ROLE |
| BC_ACCOUNTCONTACT    | BC_CONTACT                  | LEFT      | BC_CONTACT.ID = BC_ACCOUNTCONTACT.CONTACTID     |
| BC_CONTACT           | BC_ADDRESS                  | LEFT      | BC_ADDRESS.ID = BC_CONTACT.PRIMARYADDRESSID     |
| BC_ADDRESS           | BCTL_STATE                  | LEFT      | BCTL_STATE.ID = BC_ADDRESS.STATE                |

## Transformations

- **Derived Column**: BatchID assignment (`BatchID = @[User::BatchID]`)
- **Lookup**: BeanVersion from DimBillingAccount (`Lookup on DimBillingAccount.PublicID = Source.PublicID; output EDWBeanVersion`)
- **Conditional Split**: SCD logic (`BeanVersion == EDWBeanVersion (SameBeanVersion); else (DifferentBeanVersion)`)
- **Filter**: UpdateTime delta filter (`WHERE (dt.UpdateTime >= DATEADD(D,?,GETDATE()) OR ParentAcct.UpdateTime >= DATEADD(D,?,GETDATE()) OR sz.UpdateTime >= DATEADD(D,?,GETDATE()) OR InsuredInfo.ac_UpdateTime >= DATEADD(D,?,GETDATE()) OR InsuredInfo.acr_UpdateTime >= DATEADD(D,?,GETDATE()) OR InsuredInfo.c_UpdateTime >= DATEADD(D,?,GETDATE()) OR InsuredInfo.a_UpdateTime >= DATEADD(D,?,GETDATE()))`)
- **Row Count**: Audit row counts for Source, Insert, Update, Unchange (`CNT - Source Count, CNT - Insert Count, CNT - Update Count, CNT - Unchange Count to SSIS variables`)

## Target Table

- **DIMBILLINGACCOUNT**
    - Load Type: **INSERT**
    - Source: Rows from OLE_SRC - GuideWire via CNT - Insert Count

---

### Audit Variables

- `User::SourceCount`
- `User::InsertCount`
- `User::UpdateCount`
- `User::UnChangeCount`

---

### Architecture Diagram (Textual Representation)

```
[Source Tables] --> [Joins] --> [Transformations] --> [Target Table]
```

---

### SSIS Package Metadata

- Package Name: **EDW_BC_Load_DimBillingAccount**
- Data Flow Task: **DFT - Load DimBillingAccount**
- Target Table: **DIMBILLINGACCOUNT**
- SCD Type: Conditional Split + BeanVersion comparison
- Audit: Row count variables tracked in SSIS
