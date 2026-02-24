# Trusted Workspace Access

> **Source:** [Trusted workspace access in Microsoft Fabric](https://learn.microsoft.com/en-us/fabric/security/security-trusted-workspace-access)
> Last reviewed: 2026-02-24

## What It Is

Trusted workspace access allows Fabric workloads to securely connect to **firewall-enabled Azure Data Lake Storage Gen2 (ADLS Gen2)** accounts without exposing them to the public internet. It works by using a **workspace identity** (an auto-managed service principal) to authenticate and a **resource instance rule** on the storage account to authorize specific Fabric workspaces.

The connection uses Microsoft's backbone network, not the public internet — even if the storage account has public network access disabled.

## Prerequisites

- Workspace must be associated with a **Fabric F SKU capacity** (not P SKU, not Trial)
- A **workspace identity** must be created for the workspace (see [workspace-identity.md](../03-identity/workspace-identity.md))
- The workspace identity should have **Contributor** access to the workspace
- The authenticating principal must have appropriate Azure RBAC on the storage account:
  - Storage Blob Data Contributor, Storage Blob Data Owner, or Storage Blob Data Reader (at storage account scope), OR
  - Storage Blob Delegator (at storage account scope) + folder-level access
- A **resource instance rule** must be configured on the storage account

## How Resource Instance Rules Work

Resource instance rules on ADLS Gen2 allowlist specific Azure resources (in this case, Fabric workspaces) to access the storage account even when public network access is restricted.

```json
"resourceAccessRules": [
  {
    "tenantId": "<your-tenant-id>",
    "resourceId": "/subscriptions/00000000-0000-0000-0000-000000000000/resourcegroups/Fabric/providers/Microsoft.Fabric/workspaces/<workspace-id>"
  }
]
```

**Key constraints:**
- The subscription ID in the resourceId must be `00000000-0000-0000-0000-000000000000` (a placeholder for Fabric)
- Resource instance rules for Fabric workspaces can only be created via **ARM templates or PowerShell** — the Azure portal UI does not support this
- Maximum of **200 resource instance rules** per storage account
- Up to 200 rules is also an Azure subscription limit

### Via PowerShell

```powershell
$resourceId = "/subscriptions/00000000-0000-0000-0000-000000000000/resourcegroups/Fabric/providers/Microsoft.Fabric/workspaces/<YOUR_WORKSPACE_GUID>"
$tenantId = "<YOUR_TENANT_ID>"
Add-AzStorageAccountNetworkRule -ResourceGroupName $resourceGroupName -Name $accountName -TenantId $tenantId -ResourceId $resourceId
```

## Trusted Service Exception — Use with Caution

There is an alternative to resource instance rules: the **trusted service exception** checkbox on the storage account. When checked, any Fabric workspace in the tenant with a workspace identity can access the storage account.

> Microsoft explicitly recommends against this configuration and notes that support for it may be discontinued. Use resource instance rules for specific workspace allowlisting instead.

## Supported Use Cases

| Method | Notes |
|--------|-------|
| OneLake shortcut to ADLS Gen2 | Primary use case; Spark, SQL endpoint, Pipelines, Dataflows, Semantic Models all work through the shortcut |
| Data Factory Pipeline | Direct pipeline connection to firewall-enabled ADLS Gen2 |
| T-SQL COPY statement (Warehouse) | Ingest data into a Fabric Warehouse from firewall-enabled ADLS Gen2 |
| Semantic Model (Import mode) | Connect to firewall-enabled ADLS Gen2 for model refresh |
| AzCopy | Copy data from firewall-enabled Azure Storage to OneLake; requires `--trusted-microsoft-suffixes "fabric.microsoft.com"` |

**Not supported via trusted workspace access (use managed private endpoints instead):**
- Spark notebooks/jobs directly accessing storage — must use managed private endpoints

## Authentication Methods

- **Organizational account** — for user-context connections
- **Service principal** — for automated/service connections
- **Workspace identity** — for workspace-level trusted access

> Connections to firewall-enabled storage accounts show status **Offline** in Manage Connections and Gateways. This is expected behavior and does not indicate a problem.

## Shortcut Behavior

- Preexisting shortcuts created **before October 10, 2023** do not support trusted workspace access
- Preexisting shortcuts in workspaces meeting the prerequisites will automatically start supporting trusted service access
- Use the DFS URL format: `https://<StorageAccountName>.dfs.core.windows.net`

## Limitations

| Limitation | Detail |
|-----------|--------|
| SKU requirement | F SKU only; not supported on Trial capacities |
| Pipelines cannot write to OneLake table shortcuts | Temporary limitation |
| Cross-tenant requests | Not supported |
| Connections for trusted access reused in other workspaces | May not work |
| If your org has Conditional Access for workload identities that includes all service principals | Trusted workspace access won't work — must exclude specific Fabric workspace identities |
| If workspace migrated to non-F SKU capacity | Trusted workspace access stops working after 1 hour |

## Connection Management Quirks

- Connections for trusted workspace access can be created in **Manage connections and gateways**, but workspace identity is the only supported auth method
- Test connection will fail if organizational account or service principal is used as auth method in this UI
- If semantic models use personal cloud connections, only workspace identity is supported for trusted access

## Community Findings

> **Source:** r/MicrosoftFabric (Jul 2025)

A community comparison of outbound connectivity options ranked TWA as "easiest but least secure." This framing is worth examining:

- **What's accurate**: The storage account endpoint remains publicly addressable even with TWA. The resource instance rule opens access to the workspace identity, but an attacker who compromised the workspace identity could reach the storage account from anywhere. This is meaningfully different from a private endpoint, where the storage account is not reachable from the public internet at all.
- **What's misleading**: "Least secure" implies weak authentication. TWA authentication is Entra ID / workspace identity — the same credential model as MPEs. The risk difference is network exposure, not authentication strength.
- **The right framing**: TWA is the lowest-friction option but does not provide network-layer isolation. For environments where the storage account must not be publicly addressable under any circumstances, TWA is insufficient — use managed private endpoints (Spark) or VNet data gateway (Pipelines/Dataflows).
- **Scope reminder**: TWA only applies to **ADLS Gen2**. It is not an option for Azure SQL, Azure SQL MI, Cosmos DB, or other data sources.
