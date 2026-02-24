# Outbound Network Security — Decision Guide

> **Sources:**
> - [Security in Microsoft Fabric — Outbound network security](https://learn.microsoft.com/en-us/fabric/security/security-overview#outbound-network-security)
> - [Microsoft Fabric end-to-end security scenario](https://learn.microsoft.com/en-us/fabric/security/security-scenario)
> Last reviewed: 2026-02-24

Outbound security controls how Fabric workloads connect to **external data sources** — whether on-premises, in Azure, or in other clouds. By default, Fabric can connect to any public endpoint. For sources behind firewalls or private networks, you need one of the options below.

---

## Outbound Options at a Glance

| Option | Best For | Works With | Requires |
|--------|----------|-----------|---------|
| On-Premises Data Gateway | On-premises data, non-Azure firewall-protected sources | Dataflows Gen2, Pipelines, Power BI | Server in your network |
| VNet Data Gateway | Azure data sources behind private endpoints | Dataflows Gen2, Power BI | Azure VNet |
| Managed Private Endpoints | Azure PaaS data sources, Spark workloads | Notebooks, Lakehouse, Spark Job Definitions, Eventstream | F SKU or Trial capacity; Fabric Data Engineering in region |
| Trusted Workspace Access | Firewall-enabled ADLS Gen2 | Shortcuts, Pipelines, Semantic Models, AzCopy | Workspace Identity; F SKU; ADLS Gen2 resource instance rule |
| Service Tags | Allow Azure-hosted data sources to accept Fabric traffic | Azure NSGs, Azure Firewall | Azure resource configuration |
| Outbound Access Protection | Block all outbound by default + allowlist specific targets | Data Engineering, Data Factory, Mirroring, OneLake | F SKU; Fabric Data Engineering in region |

---

## Choosing the Right Option

### For on-premises data sources

Use the **On-Premises Data Gateway**. Install it on a server within your network. It acts as a bridge — Fabric connects through the gateway rather than directly to the data source. No inbound ports need to be opened.

> **Limitation**: On-premises data gateways do NOT work when tenant-level Private Link is enabled. You must use VNet data gateways in that case.

### For Azure data sources behind private endpoints or VNet firewalls

**Option A — Managed Private Endpoints (Spark workloads):**
Use for Notebooks, Spark Job Definitions, and Evenstream where Spark-based compute connects to the data source. A workspace admin creates managed private endpoints from workspace settings. Microsoft handles the VNet — you don't manage it yourself.

**Option B — VNet Data Gateway (Data Factory / Power BI workloads):**
Use for Dataflows Gen2 and Power BI that need to reach data in an Azure VNet. Fabric dynamically provisions containers in your VNet without requiring you to manage gateway infrastructure.

### For ADLS Gen2 storage accounts with firewall enabled

Use **Trusted Workspace Access**. Create a workspace identity, configure a resource instance rule on the storage account (via ARM template or PowerShell), and Fabric workloads (shortcuts, pipelines, semantic models) can access the storage account without exposing it publicly.

> **Limitation**: Resource instance rules for Fabric workspaces must be created via ARM templates or PowerShell — the Azure portal UI does not support this.

### For Azure SQL, Azure VMs, Azure SQL MI, or REST APIs from an Azure VNet

Use **Service Tags**. Add the appropriate Fabric service tag to your NSG or Azure Firewall rules to permit inbound connections from Fabric's IP ranges.

### To strictly control what your workspaces can connect to

Use **Outbound Access Protection** (workspace-level). This blocks all outbound connections from the workspace by default, then you explicitly allowlist:
- Managed private endpoints (for Data Engineering)
- Data connection rules (for Data Factory)

---

## Scenario Reference

| I want to... | Use this |
|-------------|----------|
| Load data from on-premises SQL Server via pipeline | On-premises data gateway + pipeline copy activity |
| Load data from on-premises systems via low-code | On-premises data gateway + Dataflow Gen2 |
| Connect to Azure SQL behind private endpoint (no gateway infra to manage) | VNet data gateway + Dataflow Gen2 |
| Connect from Spark notebook to Azure SQL behind private endpoint | Managed private endpoint |
| Connect Spark to Azure Storage account with public access disabled | Managed private endpoint |
| Connect any Fabric workload to firewall-enabled ADLS Gen2 | Trusted workspace access + workspace identity |
| Migrate Azure Data Factory pipelines to load into Fabric | Lakehouse connector in existing ADF pipelines |
| Use Databricks or Synapse Spark to load into Fabric | OneLake API (ABFS driver) |
| Allow Azure SQL MI to send traffic to Fabric | Service tags (Power BI service tag) |
| Block workspaces from connecting to unapproved external resources | Outbound access protection |

---

## Documents in This Section

- [managed-vnets.md](managed-vnets.md) — Managed Virtual Networks for Spark workloads
- [managed-private-endpoints.md](managed-private-endpoints.md) — Secure outbound connections to Azure PaaS resources
- [trusted-workspace-access.md](trusted-workspace-access.md) — Secure access to firewall-enabled ADLS Gen2
- [outbound-access-protection.md](outbound-access-protection.md) — Block all outbound + allowlist specific connections
- [service-tags.md](service-tags.md) — Azure service tags for Fabric IP ranges
- [data-gateways.md](data-gateways.md) — On-premises and VNet data gateways
