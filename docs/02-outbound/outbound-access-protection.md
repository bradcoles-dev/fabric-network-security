# Workspace Outbound Access Protection

> **Source:** [Workspace outbound access protection overview](https://learn.microsoft.com/en-us/fabric/security/workspace-outbound-access-protection-overview)
> Last reviewed: 2026-02-24

## What It Is

Workspace outbound access protection lets admins **block all outbound connections from a workspace by default**, then allowlist specific approved connections. This is a data exfiltration prevention control — it ensures that workloads can only send data to pre-approved destinations.

It is the outbound complement to workspace-level private links (which control inbound access).

## How It Works

When outbound access protection is enabled:
- **All outbound connections are blocked** by default
- Workspace admins create exceptions using one of two mechanisms:
  1. **Managed private endpoints** — for Data Engineering and OneLake workloads; allows secure connections to specific Azure PaaS resources
  2. **Data connection rules** — for Data Factory workloads; allows specific connectors/endpoints

## Supported Workloads and Exception Mechanisms

| Workload | Exception Mechanism | Supported Items |
|----------|--------------------|-----------------|
| Data Engineering | Managed private endpoints | Lakehouses, Notebooks, Spark Job Definitions, Environments |
| Data Factory | Data connection rules | Dataflows Gen2 (CI/CD), Pipelines, Copy Jobs |
| Data Warehouse | Not applicable (warehouse has no configurable outbound exception) | Warehouses, SQL analytics endpoints |
| Mirrored databases | Data connection rules | Azure SQL DB, Snowflake, Cosmos DB, Azure SQL MI, PostgreSQL, SQL Server, Oracle, Google BigQuery |
| OneLake | Managed private endpoints | OneLake shortcuts |

## Cross-Workspace Connectivity

A workspace with outbound access protection enabled can connect to **another workspace in the same tenant** by combining:
1. A managed private endpoint in the source workspace pointing to the target workspace
2. Private Link service enabled on the target workspace

This enables items in the source workspace (e.g., shortcuts) to access data in the target workspace (e.g., lakehouses).

## Data Connection Rules — Connector Granularity

For Data Factory workloads, the level of allowlist control depends on the connector type:

| Connector Type | Granularity Level |
|----------------|------------------|
| Lakehouse | Workspace-level (specify which workspaces) |
| Warehouse | Workspace-level |
| Dataflows | Workspace-level |
| Fabric SQL Database | Workspace-level |
| SQL Server | Endpoint-level (specify allowed server endpoints) |
| Azure Data Lake Storage | Endpoint-level |
| Azure Blobs | Endpoint-level |
| Web / Web v2 | Endpoint-level |
| SharePoint | Endpoint-level |
| OData | Endpoint-level |
| Snowflake | Endpoint-level |
| PostgreSQL | Endpoint-level |
| Databricks | Endpoint-level |
| Amazon S3 | Endpoint-level |
| REST Service | Endpoint-level |
| HTTP Server | Endpoint-level |
| MySQL | Endpoint-level |
| Dataverse | Endpoint-level |
| Analysis Services | Endpoint-level |

> **Note**: Datamarts, KQL Database, Fabric Data Pipelines, and CopyJob connection types do **not** support workspace-level granularity for internal Fabric types.

## Feature Availability

See [feature-availability.md](../00-overview/feature-availability.md) for the per-item support matrix. Notable items in Preview:
- Pipelines, Dataflow Gen2, Copy Jobs, VNet/OPDG gateways — Preview for outbound access protection

## Known Limitations

| Limitation | Detail |
|-----------|--------|
| SKU requirement | Fabric F SKU only; F SKU Trial capacities not supported |
| Regional availability | Only where Fabric Data Engineering workloads are supported; not available in Qatar Central for data connection rules |
| Unsupported artifacts | If workspace contains unsupported items, outbound protection cannot be enabled until those items are removed |
| Semantic model in lakehouse | Not supported for existing workspaces that already contain a semantic model in a lakehouse |
| Warehouse file path queries from notebooks | Cannot query warehouse file paths using `dbo` schema from notebooks when outbound protection is enabled — use T-SQL instead |
| Fabric portal UI | Does not support enabling both inbound private links AND outbound access protection simultaneously — use the Workspaces REST API |
| OneLake Diagnostics | Not currently compatible with outbound access protection |
| External data sharing (cross-tenant) | Not supported with outbound access protection |
| `Microsoft.Network` feature registration | Must be re-registered on your Azure subscription |

## Configuring Both Inbound and Outbound

The Fabric portal UI cannot configure both workspace-level private links (inbound) and outbound access protection at the same time. To enable both simultaneously, use:

```http
POST /v1/workspaces/{workspaceId}/networkCommunicationPolicy
```

[Workspaces - Set Network Communication Policy API](https://learn.microsoft.com/en-us/rest/api/fabric/core/workspaces/set-network-communication-policy)

## Admin Visibility — Tenant-Wide API

Fabric administrators can audit all workspace network communication policies across the tenant using the **Workspaces - List Networking Communication Policies** admin API.

Requires: `Tenant.Read.All` or `Tenant.ReadWrite.All` permissions; Fabric administrator role or service principal.

Returns per workspace:
- Workspace ID
- Inbound policies (allow/deny)
- Outbound policies: public access rules, connection rules (endpoints/workspaces), gateway rules, Git access rules

## Private Link Portal Limitation for Data Connection Rules

If private links are enabled (at workspace or tenant level), the Fabric portal UI cannot configure data connection rules. Use the [Outbound Gateway Rules REST API](https://learn.microsoft.com/en-us/fabric/security/workspace-outbound-access-protection-allow-list-connector) instead.
