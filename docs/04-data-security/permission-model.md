# Permission Model

> **Source:** [Permission model — Microsoft Fabric](https://learn.microsoft.com/en-us/fabric/security/permission-model)
> Last reviewed: 2026-02-24

## Overview

Fabric has a layered permission model. A user must clear all three levels to access data:

1. **Entra Authentication** — can the user authenticate to the Microsoft Entra tenant?
2. **Fabric Access** — can the user access Microsoft Fabric?
3. **Data Security** — can the user perform the requested action on a table or file?

These levels evaluate sequentially. Microsoft Information Protection policies also apply at their appropriate level.

---

## Level 1: Workspace Roles

Workspaces are the primary access control boundary in Fabric. Four roles apply to all items within a workspace:

| Role | Delete Workspace | Add Admins | Add Members | Write Data | Create Items | Read Data |
|------|-----------------|-----------|------------|-----------|-------------|----------|
| Admin | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Member | ✗ | ✗ | ✓ | ✓ | ✓ | ✓ |
| Contributor | ✗ | ✗ | ✗ | ✓ | ✓ | ✓ |
| Viewer | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |

**Key point**: Workspace roles apply to ALL items in the workspace. There is no per-item exception at the workspace role level.

**OneLake access**: Admin, Member, and Contributor roles have OneLake access. **Viewer role does NOT have OneLake access** — they can see and run items in the Fabric portal but cannot read underlying files in OneLake directly.

---

## Level 2: Item Permissions

Item permissions allow granting access to a single Fabric item without giving workspace access. This enables the pattern of sharing a specific report or semantic model with a user who has no workspace role.

**Default sharing permission**: Grants "read" on the item — allows seeing metadata and viewing reports, but does NOT allow accessing underlying SQL data or OneLake files.

Different item types have different available permissions. Key examples:

- **Semantic model**: Read, Build (to create reports from the model)
- **Warehouse**: Read, ReadData, ReadAll, Write
- **Lakehouse**: Read, ReadAll, ReadData, Write

> **Critical understanding**: Item "read" permission and workspace "viewer" role behave differently for the same user. A workspace viewer can see all items in the workspace. Item-level read only gives access to that specific item.

### Workspace Role + Item Permission Interaction

If a user has both a workspace viewer role AND item permissions:
- Removing item permissions does NOT prevent the user from accessing the item (workspace viewer role still grants access)
- To truly remove access, you must remove both the item permission AND the workspace role

If a user has only item permissions (no workspace role):
- They can access the specific shared item via a direct link
- They cannot see the workspace or discover other items

---

## Level 3: Compute Permissions (SQL and Semantic Model)

More granular security applied within compute engines:

### SQL Analytics Endpoint
Security configured via SQL commands. Applies only to queries made through the SQL endpoint:
- Row-level security (RLS)
- Column-level security (CLS / column masking)
- Object-level security (OLS — table/view permissions)

### Semantic Model (DAX)
Security defined using DAX. Applies to:
- Users querying through the semantic model
- Power BI reports built on the semantic model

> **Important**: If RLS/OLS is defined in the SQL analytics endpoint and a DirectLake report is used, the queries automatically **fall back to DirectQuery mode** to enforce the SQL security. If you don't want this fallback, create a new lakehouse using shortcuts to the original tables and don't define SQL security there — apply security in the semantic model instead.

---

## OneLake Security (Layer 4)

OneLake has its own permission layer via **OneLake Data Access Control**. This allows creating custom roles within a lakehouse that grant read permissions only to specific tables and folders when accessing OneLake directly (not through SQL or semantic model).

- Custom roles can be assigned to users, security groups, or auto-assigned based on workspace role
- This is distinct from SQL endpoint security and semantic model security
- For details see [OneLake Data Access Control Model](https://learn.microsoft.com/en-us/fabric/onelake/security/data-access-control-model)

---

## DirectLake and RLS Considerations

DirectLake mode reads data directly from the lakehouse files. For RLS to work:

- If you add RLS to a DirectLake dataset and there's also SQL RLS defined, DirectLake falls back to DirectQuery for those tables
- To avoid DirectQuery fallback while still using RLS: use direct dataset sharing or Power BI apps (not workspace viewer), and set the credential to a **fixed identity** that has access to the files
- The fixed identity must have access to the underlying lakehouse files; RLS is enforced at the semantic model level (not at the file level)

---

## Common Patterns

### Medallion Architecture Access Control

| Layer | Who Has Access |
|-------|---------------|
| Bronze | Data engineers only (Admin/Contributor on bronze workspace) |
| Silver | Data engineers only (Admin/Contributor on silver workspace) |
| Gold (restricted lakehouse) | Data engineers + no direct sharing with analysts |
| Gold (public-facing semantic model) | Shared with analysts; RLS applied in model |
| Gold (SQL analytics endpoint with OLS/CLS) | Analysts with lakehouse item permission; SQL security limits what they see |

### External Sharing (Power BI App Pattern)

When sharing reports via a Power BI app:
- Users get access to reports through the app, not the workspace
- They cannot see the workspace or its items
- DirectLake fallback to DirectQuery happens if SQL security is defined — plan accordingly

---

## Per-Item Permission References

| Item | Permissions Doc |
|------|----------------|
| Semantic Model | [Semantic model permissions](https://learn.microsoft.com/en-us/power-bi/connect-data/service-datasets-permissions) |
| Warehouse | [Warehouse permissions](https://learn.microsoft.com/en-us/fabric/data-warehouse/share-warehouse-manage-permissions) |
| SQL Database | [SQL Database security overview](https://learn.microsoft.com/en-us/fabric/database/sql/security-overview) |
| Lakehouse | [Lakehouse sharing](https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-sharing) |
| Data Science | [ML model/experiment RBAC](https://learn.microsoft.com/en-us/fabric/data-science/models-experiments-rbac) |
| Real-Time Intelligence | [KQL security roles](https://learn.microsoft.com/en-us/azure/data-explorer/kusto/management/security-roles) |
| Mirrored Database | [Mirrored DB sharing](https://learn.microsoft.com/en-us/fabric/mirroring/share-and-manage-permissions) |
