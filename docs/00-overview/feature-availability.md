# Security Feature Availability by Fabric Item Type

> **Source:** [Microsoft Learn — Fabric security features availability](https://learn.microsoft.com/en-us/fabric/security/security-feature-availability)
> Last reviewed: 2026-02-24

This page tracks which security features are supported for each Fabric item type. This is critical for enterprise security planning — many items are **not yet supported** for key controls like workspace private links or outbound access protection.

**Features tracked:**
- **Workspace Private Links** — inbound network isolation at workspace level
- **Customer Managed Keys (CMK)** — bring-your-own encryption key
- **Outbound Access Protection** — block all outbound connections and allow specific ones

**Legend:** ✓ = GA, Preview = in preview, - = not supported/not applicable

---

## Feature Availability Matrix

| Workload | Item Type | Workspace Private Links | Customer Managed Keys | Outbound Access Protection |
|----------|-----------|------------------------|-----------------------|---------------------------|
| **Data Engineering** | Lakehouse | ✓ | ✓ | ✓ |
| | Lakehouse Shortcut | ✓ | - | Preview |
| | Lakehouse SQL Endpoint | ✓ | ✓ | ✓ |
| | Notebook | ✓ | ✓ | ✓ |
| | Spark Job Definition | ✓ | ✓ | ✓ |
| | Environment | ✓ | ✓ | ✓ |
| | Lakehouse with Schema | - | ✓ | ✓ |
| | Spark Connectors for SQL Data Warehouse | - | - | - |
| **Data Factory** | Default Semantic Model | ✓ | - | ✓ |
| | Pipeline | ✓ | ✓ | Preview |
| | Dataflow Gen1 | - | - | - |
| | Dataflow Gen2 | - | ✓ | Preview |
| | Copy Job | ✓ | ✓ | Preview |
| | Mounted Azure Data Factory | ✓ | - | - |
| | VNet data gateway | ✓ | - | Preview |
| | On-premises data gateway: Pipeline/Copy Job | ✓ | - | Preview |
| | On-premises data gateway: Dataflow Gen2 | - | - | Preview |
| | Data Workflow | - | - | - |
| | Data Build Tool job | - | - | - |
| **Data Science** | ML Model | ✓ | ✓ | |
| | Experiment | ✓ | ✓ | |
| | Data Agent | ✓ | - | |
| **Data Warehouse** | SQL Endpoint | ✓ | ✓ | ✓ |
| | Warehouse | ✓ | ✓ | ✓ |
| | Warehouse with EDPE | - | - | - |
| **Developer Experience** | API for GraphQL | - | ✓ | - |
| | Deployment Pipeline | | - | ✓ |
| | Git Integration | ✓ | - | ✓ |
| | Variable Library | ✓ | - | - |
| **Governance and Security** | Sensitivity Label | - | - | - |
| | Share item | - | - | - |
| **Graph** | Graph model | - | - | - |
| | Graph queryset | - | - | - |
| **Industry Solutions** | Healthcare data solutions | - | ✓ | - |
| | Sustainability Solution | - | ✓ | - |
| | Retail Solution | - | ✓ | - |
| **Mirroring** | Mirrored Azure SQL Database | - | - | Preview |
| | Mirrored Azure SQL Managed Instance | ✓ | - | Preview |
| | Open Mirroring | ✓ | - | - |
| | Mirrored Azure Databricks Catalog | - | - | - |
| | Mirrored Snowflake | - | - | Preview |
| | Mirrored SQL Server 2025 (Windows/Linux on-premises) | ✓ | - | Preview |
| | Mirrored SQL Server 2016–2022 | - | - | - |
| | Mirrored Dataverse | - | - | - |
| | Mirrored SAP | - | - | - |
| | Mirrored Azure Cosmos DB | ✓ | - | Preview |
| | Mirrored Azure Database for PostgreSQL | - | - | Preview |
| | Mirrored Google BigQuery | - | - | Preview |
| | Mirrored Oracle | | - | Preview |
| **Native Databases** | SQL DB in Fabric | | Preview | - |
| | Cosmos DB | | | - |
| | Snowflake database | - | - | - |
| **OneLake** | Shortcut | ✓ | - | - |
| **Power BI** | Power BI Report | - | - | - |
| | Dashboard | - | - | - |
| | Scorecard | - | - | - |
| | Semantic Model | - | - | - |
| | Streaming dataflow | - | - | - |
| | Streaming dataset | - | - | - |
| | Paginated Report | - | - | - |
| | Datamart | - | - | - |
| | Exploration | - | - | - |
| | Org App | - | - | - |
| | Metric Set | - | - | - |
| **Real-Time Intelligence** | KQL Queryset | ✓ | Preview | - |
| | Activator | ✓ | - | - |
| | Eventhouse/KQL DB | ✓ | Preview | |
| | Eventstream | ✓ | | - |
| | Real-Time Dashboard | ✓ | Preview | - |
| | Anomaly detector | - | - | - |
| | Digital Twin Builder | - | - | - |
| | Event Schema Set | - | - | - |
| | Map | - | - | - |
| **Uncategorized** | Operations Agent | - | - | - |

---

## Enterprise Planning Notes

### Power BI items have no network isolation support

The entire Power BI workload — reports, semantic models, dashboards, paginated reports, scorecards — shows `-` for workspace private links and outbound access protection. This is a significant gap for organizations expecting full network isolation across all Fabric experiences.

**Implication**: If you enable tenant-level Private Link with Block Public Internet Access, Power BI report publishing, export (PDF/PowerPoint), email subscriptions, Copilot, and modern usage metrics are all impacted or broken. See [private-links-tenant.md](../01-inbound/private-links-tenant.md) for the full list of limitations.

### Dataflow Gen1 has no support across all three features

Dataflow Gen1 is not supported for any of the three tracked features. Organizations using Dataflow Gen1 cannot include those items in network-isolated workspaces.

### Many mirroring scenarios are still Preview for outbound protection

Most mirrored database types only have Preview support for outbound access protection. SQL Server 2016–2022, Dataverse, SAP, Azure SQL Database (non-Managed Instance), and Databricks Catalog have no outbound access protection support at all.

### Workspace Private Links require F SKU

Workspace-level Private Links are only supported on Fabric F SKU capacities. P SKU (Premium) and trial capacities are not supported.

### CMK is still Preview for several RTI items

KQL Queryset, Eventhouse/KQL DB, and Real-Time Dashboard have CMK in Preview. Plan accordingly if RTI workloads are in scope for CMK requirements.

---

> For the most current status, refer to the [Microsoft Fabric Roadmap](https://roadmap.fabric.microsoft.com/?product=administration%2Cgovernanceandsecurity).
