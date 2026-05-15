# AlphaPak Industries — D365 F&O Development Project
### Context Document for AI-Assisted Development (Cursor)

---

## Project Overview

**Company:** AlphaPak Industries — a circuit board manufacturer based in Multan, Pakistan.
**D365 Environment:** USMF demo company (standard Microsoft demo data)
**Visual Studio Model:** `AlphaPakCustomizations`
**Purpose:** One progressive model that grows task by task — nothing is throwaway, every task builds on the previous one.

**Business Story:**
- AlphaPak buys raw materials (copper, semiconductors) from vendors
- Manufactures circuit boards using Bill of Materials
- Sells finished boards to corporate customers
- Needs custom fields, validations, reports, and integrations on top of standard D365 F&O

---

## Technical Stack

- **Language:** X++ (D365 F&O)
- **IDE:** Visual Studio with D365 F&O development tools
- **Base Layer:** USR layer (custom model on top of SYS)
- **Demo Data Company:** USMF
- **Reporting:** SSRS (SQL Server Reporting Services) with X++ Data Provider
- **Integration:** OData, DMF (Data Management Framework)
- **Analytics:** Power BI connected via OData

---

## Key D365 Concepts Used Across Tasks

- **Table Extension:** Adds fields to existing D365 tables without modifying base code. Done via AOT > Table Extensions.
- **Form Extension:** Adds UI controls/tabs to existing forms without modifying base forms.
- **Display Method:** A method on a table that returns a computed value shown on a form — not stored in DB.
- **Chain of Command (CoC):** Wraps existing method logic. Uses `[ExtensionOf(classStr/tableStr(X))]`, `final class`, and `next methodName()` to call original. Logic goes before or after `next`.
- **Event Handler:** Reacts to data events (insert, update, delete) using `[DataEventHandler(tableStr(X), DataEventType::Inserted)]`. Does not use `next`. Fire-and-forget.
- **SSRS Report:** Consists of 4 parts — DP class (extends `SRSReportDataProviderBase`, implements `processReport()`), Data Contract class (report parameters), Controller class (launches report), RDL file (layout in SSRS designer).
- **TempDB Table:** Used in SSRS for large datasets — supports joins, better performance than InMemory.
- **Data Entity:** Abstraction layer over one or more tables. Used for DMF import/export and OData. Created in AOT as a new Data Entity object.
- **DMF (Data Management Framework):** Uses data entities to import/export via Excel, CSV, XML. Has staging tables for validation before committing.
- **OData:** Exposing a data entity publicly for external consumption. Enabled via `IsPublic = Yes` and `PublicEntityName` on the entity.
- **SysOperation Framework:** Modern batch job pattern. Has 3 classes — Controller (UI/dialog), DataContract (parameters), Service (actual logic). Replaces old RunBase framework.

---

## Task 1 — Table Extension on InventTable

**Topic:** Table Extension
**Builds:** Foundation for all 15 tasks — these fields appear in reports, forms, entities, batch jobs

**What to create:**
- A Table Extension on `InventTable` named `InventTable.AlphaPak`

**Fields to add:**

| Field Name | EDT / Base Type | Label | Description |
|---|---|---|---|
| `AlphaPakReorderQty` | `Qty` (existing EDT) | Reorder Quantity | Minimum stock level before reorder is needed |
| `AlphaPakLocalSupplier` | `NoYes` (existing enum) | Local Supplier | Flags if this item is sourced from a local Pakistani vendor |

**Notes:**
- Use existing EDT `Qty` — do not create a new EDT
- Use existing enum `NoYes` — do not create a new enum
- Field group: add both fields to a new field group called `AlphaPakDetails` on the extension
- No custom enums or EDTs needed for this task

**Output:** `InventTable.AlphaPak` table extension compiles clean in `AlphaPakCustomizations` model

---

## Task 2 — Form Extension on Released Products

**Topic:** Form Extension
**Depends on:** Task 1 (fields must exist on InventTable before binding to form)
**Builds toward:** Task 3 (display method added to this same form)

**What to create:**
- A Form Extension on `EcoResProductDetailsExtended` (this is the Released Products form in D365 F&O)
- Named `EcoResProductDetailsExtended.AlphaPak`

**Changes to make on the form extension:**
- Add a new FastTab (Tab Page control) named `AlphaPakDetailsTab` with label "AlphaPak Details"
- Inside the FastTab, add two controls bound to the Task 1 fields:
  - `AlphaPakReorderQty` — bound to `InventTable.AlphaPakReorderQty`
  - `AlphaPakLocalSupplier` — bound to `InventTable.AlphaPakLocalSupplier`

**Notes:**
- The Released Products form uses `InventTable` as its datasource — bindings will work directly
- FastTab should appear after the standard "General" tab
- Do not modify base form — extension only

**Output:** Open Released Products in USMF, find any item, see "AlphaPak Details" tab with both fields visible and editable

---

## Task 3 — Display Method on Form

**Topic:** Display Method
**Depends on:** Task 1 (ReorderQty field), Task 2 (form where method will be shown)
**Builds toward:** Task 6 (SSRS report uses same reorder logic)

**What to create:**
- A class extension on `InventTable` named `InventTable_AlphaPak_Display`
- Add a `display` method that returns a reorder status string

**Method logic:**
```xpp
[ExtensionOf(tableStr(InventTable))]
final class InventTable_AlphaPak_Display
{
    display AlphaPakReorderStatus alphaReorderStatus()
    {
        InventSum inventSum;
        select sum(PhysicalInvent) from inventSum
            where inventSum.ItemId == this.ItemId;

        if (this.AlphaPakReorderQty > 0 && inventSum.PhysicalInvent < this.AlphaPakReorderQty)
            return "⚠ Reorder Needed";
        else if (this.AlphaPakReorderQty == 0)
            return "No threshold set";
        else
            return "Stock OK";
    }
}
```

**Form changes (Task 2 extension):**
- Add a String control on the AlphaPak FastTab named `ReorderStatusDisplay`
- Bind it to the display method `alphaReorderStatus` from `InventTable` datasource
- Set control as read-only

**Notes:**
- `display` keyword means value is computed, not stored
- `AlphaPakReorderStatus` — use existing `str` type or a suitable EDT, do not create custom EDT
- `InventSum` is the standard D365 on-hand inventory table

**Output:** On Released Products form, AlphaPak tab shows live "⚠ Reorder Needed" or "Stock OK" label per item

---

## Task 4 — CoC on InventTable.validateWrite

**Topic:** Chain of Command
**Depends on:** Task 1 (fields being validated)
**Builds toward:** Task 9 (DMF import will trigger this validation)

**What to create:**
- A class extension named `InventTable_AlphaPak_Extension`
- CoC on `InventTable.validateWrite()`

**Business rule:**
If `AlphaPakLocalSupplier` is set to `Yes`, then `AlphaPakReorderQty` must be greater than 0. If not, block the save with an error.

**Code structure:**
```xpp
[ExtensionOf(tableStr(InventTable))]
final class InventTable_AlphaPak_Extension
{
    public boolean validateWrite()
    {
        boolean ret;

        ret = next validateWrite(); // run original validation first

        if (ret && this.AlphaPakLocalSupplier == NoYes::Yes
                && this.AlphaPakReorderQty <= 0)
        {
            ret = checkFailed("AlphaPak: Local supplier items must have a Reorder Quantity defined.");
        }

        return ret;
    }
}
```

**Notes:**
- `next validateWrite()` must always be called — runs the original D365 validation first
- `checkFailed()` blocks the save and shows error in infolog
- Logic goes AFTER `next` so base validation runs first
- Return `ret` always — do not return hardcoded `true` or `false`

**Output:** Try saving a Released Product with LocalSupplier = Yes and ReorderQty = 0 → error blocks save. Set ReorderQty > 0 → saves fine.

---

## Task 5 — Event Handler on InventTable

**Topic:** Event Handler
**Depends on:** Task 1 (tracking changes to ReorderQty field)
**Builds toward:** Task 11 (batch job monitors same field)

**What to create:**
- Static event handler class named `InventTable_AlphaPak_EventHandler`
- Handle the `onUpdated` data event on `InventTable`

**Business rule:**
When any `InventTable` record is updated, check if `AlphaPakReorderQty` changed. If it did, log an info message showing old and new value.

**Code structure:**
```xpp
public static class InventTable_AlphaPak_EventHandler
{
    [DataEventHandler(tableStr(InventTable), DataEventType::Updated)]
    public static void InventTable_onUpdated(Common sender, DataEventArgs e)
    {
        InventTable inventTable = sender as InventTable;
        InventTable orig = inventTable.orig();

        if (inventTable.AlphaPakReorderQty != orig.AlphaPakReorderQty)
        {
            info(strFmt("AlphaPak: Reorder Qty changed for item %1 — Old: %2, New: %3",
                inventTable.ItemId,
                orig.AlphaPakReorderQty,
                inventTable.AlphaPakReorderQty));
        }
    }
}
```

**Notes:**
- Event handlers are `static` — no `next`, no return value
- `orig()` returns the original record before update — use this to compare old vs new value
- `DataEventType::Updated` fires after the record is saved
- This is different from CoC — CoC wraps a method, event handler reacts to a data event

**Output:** Change ReorderQty on any item in Released Products → infolog shows old and new value

---

## Task 6 — Basic SSRS Report: Reorder Alert Report

**Topic:** SSRS — Full architecture (DP + Contract + Controller + RDL)
**Depends on:** Task 1 (ReorderQty field), Task 3 (same reorder logic now in a report)
**Builds toward:** Task 7 (same architecture, more complex)

**Business need:** Print/view a list of all AlphaPak items where current on-hand stock is below `AlphaPakReorderQty`

**What to create — 4 components:**

### 1. Temp Table: `AlphaPakReorderTmp`
```xpp
// TempDB table (not InMemory — needed for joins in SSRS)
// Fields:
// - ItemId (ItemId EDT)
// - ItemName (Name EDT)
// - ReorderQty (Qty EDT)
// - OnHandQty (Qty EDT)
// - LocalSupplier (NoYes)
// TableType property = TempDB
```

### 2. Data Contract: `AlphaPakReorderContract`
```xpp
[DataContractAttribute]
public class AlphaPakReorderContract
{
    boolean localSupplierOnly;

    [DataMemberAttribute('LocalSupplierOnly')]
    public boolean parmLocalSupplierOnly(boolean _val = localSupplierOnly)
    {
        localSupplierOnly = _val;
        return localSupplierOnly;
    }
}
```

### 3. Data Provider: `AlphaPakReorderDP`
```xpp
[SRSReportParameterAttribute(classStr(AlphaPakReorderContract))]
public class AlphaPakReorderDP extends SRSReportDataProviderBase
{
    AlphaPakReorderTmp tmpTable;

    [SRSReportDataSetAttribute('AlphaPakReorderTmp')]
    public AlphaPakReorderTmp getReorderData()
    {
        select * from tmpTable;
        return tmpTable;
    }

    public void processReport()
    {
        AlphaPakReorderContract contract = this.parmDataContract();
        InventTable inventTable;
        InventSum inventSum;

        while select inventTable
            where (!contract.parmLocalSupplierOnly() || inventTable.AlphaPakLocalSupplier == NoYes::Yes)
            join sum(PhysicalInvent) from inventSum
                where inventSum.ItemId == inventTable.ItemId
        {
            if (inventTable.AlphaPakReorderQty > 0
                && inventSum.PhysicalInvent < inventTable.AlphaPakReorderQty)
            {
                tmpTable.ItemId = inventTable.ItemId;
                tmpTable.ItemName = inventTable.itemName();
                tmpTable.ReorderQty = inventTable.AlphaPakReorderQty;
                tmpTable.OnHandQty = inventSum.PhysicalInvent;
                tmpTable.LocalSupplier = inventTable.AlphaPakLocalSupplier;
                tmpTable.insert();
            }
        }
    }
}
```

### 4. Controller: `AlphaPakReorderController`
```xpp
public class AlphaPakReorderController extends SrsReportRunController
{
    public static void main(Args _args)
    {
        AlphaPakReorderController controller = new AlphaPakReorderController();
        controller.parmReportName(ssrsReportStr(AlphaPakReorderReport, Report));
        controller.parmArgs(_args);
        controller.startOperation();
    }
}
```

**RDL:** Design in Visual Studio SSRS designer — table layout showing ItemId, ItemName, OnHandQty, ReorderQty, LocalSupplier columns. Add report parameter for LocalSupplierOnly checkbox.

**Notes:**
- Always use TempDB tables in SSRS — not InMemory. InMemory cannot be joined in queries.
- DP class extends `SRSReportDataProviderBase` and implements `processReport()`
- One dataset per TempDB table
- Menu item (Output type) needed to launch the report from D365 UI

**Output:** Run report from D365 menu — shows all items needing reorder, filterable by local supplier

---

## Task 7 — Advanced SSRS Report: Vendor Aging

**Topic:** SSRS — TempDB, aging buckets, multi-table query, parameters
**Depends on:** Task 1 (LocalSupplier flag links items to vendors), Task 6 (same SSRS architecture)
**Builds toward:** Task 12 (AP module vendors are the same vendors here)

**Business need:** Show outstanding vendor invoices aged into 30/60/90/90+ day buckets — focused on vendors who supply LocalSupplier items from Task 1

**What to create:**

### TempDB Table: `AlphaPakVendAgingTmp`
Fields: `VendAccount`, `VendName`, `InvoiceId`, `InvoiceDate`, `DueDate`, `Amount`, `DaysOverdue`, `Bucket30`, `Bucket60`, `Bucket90`, `BucketOver90`

### Aging Logic in DP `processReport()`:
```xpp
// Join VendTrans + VendTable
// Filter: open transactions only (VendTrans.TransDate not settled)
// Calculate DaysOverdue = today() - DueDate
// Populate bucket fields:
//   Bucket30 = amount if DaysOverdue <= 30
//   Bucket60 = amount if DaysOverdue 31-60
//   Bucket90 = amount if DaysOverdue 61-90
//   BucketOver90 = amount if DaysOverdue > 90
```

### Contract parameters:
- `FromDate` / `ToDate` (TransDate filter)
- `LocalSupplierItemsOnly` (NoYes) — when Yes, only show vendors linked to Task 1 LocalSupplier items

**RDL layout:** Group by vendor, subtotal per bucket column, grand total row at bottom

**Notes:**
- `VendTrans` is the vendor transaction table
- `VendTable` has vendor name and account details
- Link to Task 1: join through `VendTable` → `InventTable` where `AlphaPakLocalSupplier = Yes`

**Output:** Parameterized aging report showing vendor balances in aging buckets

---

## Task 8 — Custom Data Entity

**Topic:** Data Entity
**Depends on:** Task 1 (entity must expose extension fields)
**Builds toward:** Task 9 (DMF uses this entity), Task 10 (OData uses this entity)

**What to create:**
- Data Entity named `AlphaPakInventItemEntity`
- Primary datasource: `InventTable`
- Secondary datasource: `EcoResProduct` (for item name)

**Fields to expose:**

| Entity Field | Maps To | Notes |
|---|---|---|
| `ItemId` | `InventTable.ItemId` | Key field |
| `ItemName` | `EcoResProduct.Name` | From joined table |
| `AlphaPakReorderQty` | `InventTable.AlphaPakReorderQty` | Task 1 field |
| `AlphaPakLocalSupplier` | `InventTable.AlphaPakLocalSupplier` | Task 1 field |
| `ItemGroupId` | `InventTable.ItemGroupId` | Standard field |

**Entity properties:**
- `IsPublic = Yes`
- `PublicEntityName = AlphaPakInventItem`
- `PublicCollectionName = AlphaPakInventItems`
- `EntityCategory = Master`

**Notes:**
- Data entity sits in AOT under Data Model > Data Entities
- Must add to a Data Entity project and sync
- Staging table is auto-generated by D365 when entity is created

**Output:** Entity visible in Data Management workspace. Can be selected as import/export format.

---

## Task 9 — DMF Import

**Topic:** DMF (Data Management Framework)
**Depends on:** Task 8 (entity must exist), Task 4 (CoC validation will fire during import)
**Builds toward:** Task 11 (items imported here are used in batch job)

**What to do:**
1. Go to Data Management workspace in USMF
2. Create a new Import project using `AlphaPakInventItemEntity`
3. Prepare an Excel file with 20 items:
   - 10 items with `AlphaPakLocalSupplier = Yes` and valid `ReorderQty`
   - 5 items with `AlphaPakLocalSupplier = Yes` and `ReorderQty = 0` (should fail Task 4 CoC)
   - 5 items with `AlphaPakLocalSupplier = No`
4. Import and observe staging table results
5. Fix errors and re-import

**What to learn:**
- How staging tables catch errors before committing to live tables
- How CoC fires even during DMF import
- Error handling and re-import flow in DMF

**Output:** 15 clean items imported, 5 failed with Task 4 CoC error visible in staging errors

---

## Task 10 — OData Exposure

**Topic:** OData
**Depends on:** Task 8 (entity must be public)
**Builds toward:** Task 15 (Power BI connects via this OData endpoint)

**What to do:**
1. Confirm `AlphaPakInventItemEntity` has `IsPublic = Yes` and `PublicEntityName` set (Task 8)
2. Deploy and sync the entity
3. Test OData endpoint in browser:
   `https://[your-d365-url]/data/AlphaPakInventItems`
4. Connect Power BI Desktop:
   - Get Data → OData Feed
   - Enter D365 OData URL
   - Select `AlphaPakInventItems` entity
   - Load and verify Task 1 fields appear

**Notes:**
- OData URL format: `https://[environment].cloudax.dynamics.com/data/`
- Authentication: Azure AD OAuth
- All public entities are automatically available on the OData endpoint after deployment

**Output:** Power BI can pull live AlphaPak item data including ReorderQty and LocalSupplier fields

---

## Task 11 — SysOperation Batch Job: Nightly Reorder Alert

**Topic:** SysOperation Framework (modern batch job)
**Depends on:** Task 1 (ReorderQty field), Task 9 (items exist in system from import), Task 12 (creates PO against vendors)
**Builds toward:** Task 12 (batch creates purchase orders for AP module)

**Business rule:** Nightly job finds all items where on-hand stock < `AlphaPakReorderQty`. For `LocalSupplier` items, auto-creates a draft Purchase Order.

**What to create — 3 classes:**

### 1. DataContract: `AlphaPakReorderBatchContract`
```xpp
[DataContractAttribute]
public class AlphaPakReorderBatchContract
{
    boolean localSupplierOnly;

    [DataMemberAttribute]
    public boolean parmLocalSupplierOnly(boolean _v = localSupplierOnly)
    {
        localSupplierOnly = _v;
        return localSupplierOnly;
    }
}
```

### 2. Service: `AlphaPakReorderBatchService`
```xpp
public class AlphaPakReorderBatchService
{
    public void runReorderCheck(AlphaPakReorderBatchContract _contract)
    {
        InventTable inventTable;
        InventSum inventSum;

        while select inventTable
            where (!_contract.parmLocalSupplierOnly()
                   || inventTable.AlphaPakLocalSupplier == NoYes::Yes)
            join sum(PhysicalInvent) from inventSum
                where inventSum.ItemId == inventTable.ItemId
        {
            if (inventTable.AlphaPakReorderQty > 0
                && inventSum.PhysicalInvent < inventTable.AlphaPakReorderQty)
            {
                // Log alert
                info(strFmt("Reorder alert: Item %1 — On Hand: %2, Threshold: %3",
                    inventTable.ItemId,
                    inventSum.PhysicalInvent,
                    inventTable.AlphaPakReorderQty));

                // Task 12 hook: create draft PO for local supplier items
                // (PO creation logic added in Task 12)
            }
        }
    }
}
```

### 3. Controller: `AlphaPakReorderBatchController`
```xpp
class AlphaPakReorderBatchController extends SysOperationServiceController
{
    public static void main(Args _args)
    {
        AlphaPakReorderBatchController controller =
            new AlphaPakReorderBatchController(
                classStr(AlphaPakReorderBatchService),
                methodStr(AlphaPakReorderBatchService, runReorderCheck),
                SysOperationExecutionMode::Synchronous);
        controller.startOperation();
    }
}
```

**Output:** Run job manually from menu → infolog lists all items needing reorder. Schedule as batch for nightly run.

---

## Task 12 — AP Module: Vendor Extension + Purchase Order

**Topic:** Accounts Payable — VendTable extension, form extension, PO creation
**Depends on:** Task 1 (LocalSupplier flag), Task 11 (batch job hooks into PO creation here)
**Builds toward:** Task 15 (vendor data in Power BI dashboard)

**What to create:**

### Table Extension: `VendTable.AlphaPak`
Fields:
- `AlphaPakSupplierCategory` — use `str` or existing GroupId EDT. Categories: Local Raw, Local Packaging, International

### Form Extension: `VendTableListPage.AlphaPak`
- Add `AlphaPakSupplierCategory` field to vendor list form

### CoC on PurchTable:
```xpp
[ExtensionOf(tableStr(PurchTable))]
final class PurchTable_AlphaPak_Extension
{
    public boolean validateWrite()
    {
        boolean ret = next validateWrite();

        if (ret && !this.PurchName)
        {
            ret = checkFailed("AlphaPak: Purchase order must have a description.");
        }

        return ret;
    }
}
```

### Complete Task 11 hook — PO creation:
In `AlphaPakReorderBatchService`, add logic to create a draft `PurchTable` record for items where `AlphaPakLocalSupplier = Yes`, linked to their primary vendor from `InventItemVendorCust` table.

**Output:** Vendors have category field. Batch job from Task 11 now auto-creates draft POs for local supplier items.

---

## Task 13 — AR Module: Customer Extension + Sales Validation

**Topic:** Accounts Receivable — CustTable extension, form extension, SalesTable CoC
**Depends on:** Task 1 (ReorderQty — sales validation checks stock won't drop below threshold)
**Builds toward:** Task 15 (customer data in Power BI dashboard)

**What to create:**

### Table Extension: `CustTable.AlphaPak`
Fields:
- `AlphaPakAccountManager` — use `Name` EDT (existing)
- `AlphaPakCreditRiskScore` — use `integer` base type, range 1–10

### Form Extension: `CustTableListPage.AlphaPak`
- Add new FastTab "AlphaPak Details" with both fields

### CoC on SalesTable — stock check:
```xpp
[ExtensionOf(tableStr(SalesTable))]
final class SalesTable_AlphaPak_Extension
{
    public boolean validateWrite()
    {
        boolean ret = next validateWrite();

        // Warn if customer credit risk score is high
        CustTable custTable = CustTable::find(this.CustAccount);
        if (ret && custTable.AlphaPakCreditRiskScore >= 8)
        {
            warning(strFmt("AlphaPak: Customer %1 has a high credit risk score (%2). Proceed with caution.",
                this.CustAccount,
                custTable.AlphaPakCreditRiskScore));
        }

        return ret;
    }
}
```

**Output:** Customers have account manager and risk score. Creating a sales order for a high-risk customer shows warning.

---

## Task 14 — BOM Setup + Validation

**Topic:** Bill of Materials — BOM structure, components, CoC on BOMTable
**Depends on:** Task 9 (items imported via DMF are the BOM components)
**Builds toward:** Task 15 (production data in Power BI)

**What to do in USMF:**
1. Go to Product Information Management > Bills of Materials
2. Create BOM for a finished goods item (e.g. "AlphaPak Circuit Board Type-A")
3. Add 4–5 raw material components from items imported in Task 9
4. Set quantities per component
5. Activate the BOM

**CoC on BOMTable:**
```xpp
[ExtensionOf(tableStr(BOMTable))]
final class BOMTable_AlphaPak_Extension
{
    public boolean validateWrite()
    {
        boolean ret = next validateWrite();

        // AlphaPak rule: BOM must have at least 2 components before activation
        BOMVersion bomVersion;
        BOM bom;
        int lineCount;

        select count(RecId) from bom
            where bom.BOMId == this.BOMId;

        lineCount = int642int(bom.RecId);

        if (ret && this.Active == NoYes::Yes && lineCount < 2)
        {
            ret = checkFailed("AlphaPak: A BOM must have at least 2 components before it can be activated.");
        }

        return ret;
    }
}
```

**Output:** BOM created with Task 9 items as components. Activating a BOM with < 2 lines is blocked.

---

## Task 15 — Power BI Executive Dashboard

**Topic:** Power BI via OData
**Depends on:** Task 10 (OData entity), Tasks 12, 13, 14 (AP, AR, BOM data)
**Builds toward:** Final portfolio deliverable

**What to connect:**
- `AlphaPakInventItems` (Task 10 OData entity) — inventory + reorder data
- Standard `VendInvoiceTransactions` entity — AP aging data (used in Task 7 report)
- Standard `CustomerTransactions` entity — AR open balances
- Standard `ProductionOrders` entity — production status

**Dashboards to build in Power BI:**

| Page | Shows |
|---|---|
| Inventory | Items by reorder status, local vs international supplier split |
| Payables | Vendor aging buckets (from Task 7 logic), top vendors by spend |
| Receivables | Customer balances, high risk customers (AlphaPakCreditRiskScore >= 8) |
| Production | BOM components, production order status |

**Notes:**
- Relationships in Power BI: link on `ItemId`, `VendAccount`, `CustAccount`
- Use AlphaPak fields (ReorderQty, LocalSupplier, CreditRiskScore) as slicers

**Output:** Live Power BI dashboard connected to USMF, refreshable, showing AlphaPak business health across all modules

---

## Summary — What Each Task Teaches

| # | Task | Primary Skill |
|---|---|---|
| 1 | InventTable Extension | Table Extension, EDT reuse |
| 2 | Released Products Form | Form Extension, FastTab, control binding |
| 3 | Display Method | Display method, computed fields, InventSum query |
| 4 | validateWrite CoC | CoC, `next`, `checkFailed`, before/after logic |
| 5 | onUpdated Event Handler | Data event handler, `orig()`, CoC vs Handler |
| 6 | Reorder SSRS Report | DP class, Contract, Controller, TempDB table, RDL |
| 7 | Vendor Aging SSRS | Multi-table query, aging logic, report parameters |
| 8 | Custom Data Entity | Data entity, field mapping, OData properties |
| 9 | DMF Import | DMF, staging tables, error handling, import flow |
| 10 | OData Exposure | OData endpoint, Power BI connection |
| 11 | Batch Job | SysOperation — Contract, Service, Controller |
| 12 | AP Module | VendTable extension, PurchTable CoC, PO creation |
| 13 | AR Module | CustTable extension, SalesTable CoC, risk logic |
| 14 | BOM | BOM structure, BOMTable CoC, component validation |
| 15 | Power BI Dashboard | OData + Power BI, relationships, DAX slicers |

---

*Model: AlphaPakCustomizations | Company: USMF | Developer: Talha Azhar*
