# Managed Private Endpoints

> **Sources:**
> - [Overview of managed private endpoints for Microsoft Fabric](https://learn.microsoft.com/en-us/fabric/security/security-managed-private-endpoints-overview)
> - [Blog: Managed Private Endpoints Public Preview](https://blog.fabric.microsoft.com/en-us/blog/introducing-managed-private-endpoints-for-microsoft-fabric-in-public-preview/) — Feb 28, 2024
> - [Blog: GA of Private Links, Trusted Workspace Access, and Managed Private Endpoints](https://blog.fabric.microsoft.com/en-us/blog/announcing-general-availability-of-fabric-private-links-trusted-workspace-access-and-managed-private-endpoints/) — May 31, 2024
> - [Blog: APIs for Managed Private Endpoint now available](https://blog.fabric.microsoft.com/en-US/blog/apis-for-managed-private-endpoint-are-now-available/) — Oct 29, 2024
> - [Blog: Eventstream MPE GA](https://blog.fabric.microsoft.com/en-us/blog/secure-data-streaming-with-private-endpoints-now-generally-available-in-eventstream) — Jul 16, 2025
> - [Blog: Connecting to on-premises via Private Link Service](https://blog.fabric.microsoft.com/en-US/blog/securely-accessing-on-premises-data-with-fabric-data-engineering-workloads/) — Oct 22, 2025
> Last reviewed: 2026-02-24

## Feature Timeline

| Date | Event |
|------|-------|
| Feb 28, 2024 | Managed Private Endpoints enter **Public Preview** — Fabric Data Engineering only; requires **F64 or higher** SKU |
| May 31, 2024 | Managed Private Endpoints go **GA** — SKU requirement relaxed to **all F SKUs** (including trial); Azure Event Hub and IoT Hub added as supported sources |
| Oct 29, 2024 | **REST APIs** for managed private endpoints released (Create, Delete, Get, List) |
| Oct 7, 2024 | Eventstream MPE support enters **Preview** (Event Hub, IoT Hub sources) |
| Jul 16, 2025 | Eventstream MPE goes **GA** — improved diagnostics, expanded regional coverage, new UI indicator for secure connections |
| Oct 22, 2025 | MPE via **Private Link Service** (FQDN-based) announced — enables on-premises and non-Azure connectivity |

> **SKU change note**: At preview launch, MPE required F64 or higher. At GA (May 2024) this was relaxed to all F SKUs including trial. Any pre-GA documentation or community content referencing an F64 minimum is outdated.

## What They Are

Managed private endpoints (MPEs) are Azure Private Link connections that workspace admins create from within Fabric workspace settings. They enable Fabric workloads to connect to data sources that are behind firewalls or blocked from public internet access — **without exposing those data sources to the public network**.

Microsoft creates and manages the private endpoint. The workspace admin only needs to provide:
- Resource ID of the target data source
- Target subresource type
- Justification for the private endpoint request (used by the data source owner to approve or reject)

## What Gets Created

When a managed private endpoint is created, Fabric provisions it in a **managed VNet** dedicated to the workspace. This is infrastructure that exists entirely within Fabric — it is not visible in your Azure subscription. See [managed-vnets.md](managed-vnets.md) for the VNet creation behavior (including the ~3–5 minute Spark cold-start impact).

## Supported Workloads

- **Fabric Data Engineering**: Notebooks (Spark and Python runtimes), Lakehouses, Spark Job Definitions
- **Eventstream**: Secure connections to Azure resources via managed private endpoints (Preview)

## Supported Data Sources

Managed private endpoints support Azure PaaS services. Common examples:
- Azure Storage (Blob, ADLS Gen2)
- Azure SQL Database
- Azure SQL Managed Instance
- Azure Synapse Analytics
- Azure Cosmos DB
- Azure Key Vault
- Azure Event Hubs

> For the complete supported data source list, see [Create and use managed private endpoints](https://learn.microsoft.com/en-us/fabric/security/security-managed-private-endpoints-create#supported-data-sources).

## Approval Workflow

Creating a managed private endpoint triggers a pending approval request on the target Azure resource. The data source owner (typically the Azure resource owner) must approve the request in Azure before the connection becomes active.

## Regional Requirements

Managed private endpoints require Fabric Data Engineering (Spark-based) workload support in **both**:
- The tenant home region
- The capacity region where the workspace is assigned

If Fabric Data Engineering is not available in the region, managed private endpoint creation is blocked.

**Not supported in**: Switzerland West, West Central US

## Limitations

| Limitation | Detail |
|-----------|--------|
| SKU support | Fabric trial capacity and all F SKU capacities (not P SKU) |
| Workspace migration | Not supported across capacities in different regions once managed VNet is allocated |
| OneLake shortcuts to ADLS Gen2 / Azure Blob | Do not yet support connections via managed private endpoints |
| FQDN-based private endpoints (via Private Link Service) | Not supported via the Fabric portal UI — use REST API |
| Re-creation timing | After deleting an MPE, wait at least 15 minutes before creating a new one to the same resource |
| Cross-workspace access | Requires an MPE in the source workspace pointing to the target workspace's Private Link service |

## Managed Private Endpoints vs. Other Outbound Options

| Scenario | Managed Private Endpoints | Trusted Workspace Access | VNet Data Gateway |
|----------|--------------------------|-------------------------|-------------------|
| Spark notebooks/jobs | ✓ | — | — |
| Data Factory pipelines | — | ✓ (for ADLS Gen2) | ✓ |
| Power BI / Dataflows | — | ✓ (for ADLS Gen2) | ✓ |
| Azure SQL Database | ✓ | — | ✓ |
| ADLS Gen2 (firewall) | ✓ | ✓ | ✓ |
| On-premises data | — | — | — (use OPDG) |
| Requires infrastructure management | No | No | Partial (VNet) |
