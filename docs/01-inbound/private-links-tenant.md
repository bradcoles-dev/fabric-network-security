# Tenant-Level Private Links

> **Sources:**
> - [About private links for secure access to Fabric](https://learn.microsoft.com/en-us/fabric/security/security-private-links-overview)
> - [Set up and use a tenant-level private link](https://learn.microsoft.com/en-us/fabric/security/security-private-links-use)
> - [Blog: Azure Private Link Support for Microsoft Fabric in Public Preview](https://blog.fabric.microsoft.com/en-US/blog/announcing-azure-private-link-support-for-microsoft-fabric-in-public-preview/) — Feb 28, 2024
> - [Blog: GA of Private Links, Trusted Workspace Access, and Managed Private Endpoints](https://blog.fabric.microsoft.com/en-us/blog/announcing-general-availability-of-fabric-private-links-trusted-workspace-access-and-managed-private-endpoints/) — May 31, 2024
> Last reviewed: 2026-02-24

## Feature Timeline

| Date | Event |
|------|-------|
| Feb 28, 2024 | Tenant-level Private Links enter **Public Preview** |
| May 31, 2024 | Tenant-level Private Links go **GA** — same announcement as MPE and Trusted Workspace Access GA |

## What It Is

Tenant-level Private Links route all inbound connections to Fabric through Azure Private Link, using a private IP address in your Azure VNet. Traffic never crosses the public internet — it travels over Microsoft's backbone network. You can optionally block all public internet access to the tenant.

This is the most complete form of inbound network isolation in Fabric.

## Key Concepts

### Private Endpoint vs. Service Endpoint

Fabric implements **private endpoints**, not service endpoints. This is a single, directional technology — clients initiate connections to Fabric over the private endpoint, but Fabric cannot initiate connections back into the customer network through this mechanism. This also provides multi-tenant isolation: link identifiers prevent access to other customers' resources.

### Two Tenant Settings

The Admin portal has two relevant settings:

1. **Azure Private Link** — enables private link routing; once enabled, takes ~15 minutes to configure a tenant-specific FQDN
2. **Block Public Internet Access** — optionally blocks all public internet access to Fabric; once enabled, takes ~15 minutes to propagate

These settings work differently:

| Azure Private Link | Block Public Internet Access | Effect |
|-------------------|------------------------------|--------|
| Enabled | Disabled | Private link traffic uses private endpoint; public internet traffic still allowed |
| Enabled | Enabled | Private link traffic uses private endpoint; all public internet access blocked |
| Disabled | N/A | Standard public internet access only |

## Setup Overview

The setup involves several Azure and Fabric steps:

1. Enable **Azure Private Link** in the Fabric Admin portal
2. Create a `Microsoft.PowerBI/privateLinkServicesForPowerBI` resource via ARM template in your Azure subscription
3. Create a **Virtual Network** in Azure (subnet sizing: number of capacities + 15 IP addresses)
4. Create a **Virtual Machine** for testing (or use an existing machine in your VNet)
5. Create a **Private Endpoint** connecting to the PowerBI private link service resource
   - Private DNS zones created: `privatelink.analysis.windows.net`, `privatelink.pbidedicated.windows.net`, `privatelink.prod.powerquery.microsoft.com`
6. Test by running `nslookup` from within the VNet to verify private IP resolution
7. (Optional) Enable **Block Public Internet Access**

> **Important**: Use `Microsoft.PowerBI/privateLinkServicesForPowerBI` as the resource type in the ARM template — even though this is for Fabric, not just Power BI. This is not a typo in the docs.

## DNS Resolution

When private links are enabled, the tenant-specific FQDN resolves to private IPs within your VNet. The format is:

```
<tenant-object-id-without-hyphens>-api.privatelink.analysis.windows.net
```

Users accessing Fabric from outside the private network (or without proper DNS resolution) will fail to resolve Fabric endpoints.

## Capacity Constraints

- Fabric supports up to **450 capacities** in a tenant where Private Link is enabled
- Newly created capacities may take **up to 24 hours** to reflect in the private DNS zone and support private link
- **Trial capacity does not support Private Link**
- Private link setup is per-tenant; each private endpoint connects to one tenant only
- Cross-tenant private link (using a private endpoint in one Azure tenant to connect to Fabric in another tenant) is **not supported**

## What Is and Isn't Supported

### Supported via Private Link

- OneLake (portal, file explorer, Azure Storage Explorer, PowerShell, AzCopy)
- Warehouse and Lakehouse SQL analytics endpoint (Fabric portal + TDS via SSMS/VS Code)
- SQL Database (Fabric portal + TDS)
- Lakehouse, Notebook, Spark Job Definition, Environment (triggers managed VNet creation)
- Dataflow Gen2 (with VNet data gateway for data sources behind firewall)
- Pipelines (can load data from public endpoints into private-link-enabled lakehouse; authoring supported)
- ML Model, Experiment, Data Agent
- Eventstream (limited sources/destinations)
- Eventhouse (limited scenarios)
- Healthcare data solutions (preview)
- Fabric Events and Azure Events (with behavior changes)
- Mirrored databases (open mirroring, Cosmos DB, Azure SQL MI, SQL Server 2025 only)

### NOT Supported / Broken with Private Link

| Feature | Impact |
|---------|--------|
| On-premises data gateway | **Officially unsupported** — fails reliably when Block Public Internet Access is enabled; may work if only Azure Private Link is enabled (VM can still reach public Fabric endpoints). Microsoft recommendation: use workspace-level Private Link instead of tenant-level if OPDG must remain functional. See [data-gateways.md](../02-outbound/data-gateways.md). |
| Publish to Web (Power BI) | Not supported |
| Email subscriptions | Not supported when Block Public Internet Access is enabled |
| Export Power BI report to PDF/PowerPoint | Not supported |
| Power BI modern usage metrics | Partial data only (Report Open events only; no Report Page Views or performance data) |
| Copilot | Not currently supported in private link / closed network environments |
| Direct lake with internet-blocked tenant + datamart/dataflow as data source | Connection fails |
| Visual query in Warehouse | Does not work when Block Public Internet Access is enabled |
| OneLake regional endpoints (direct calls) | Do not work via private link |
| Tenant migration | Blocked while Private Link is enabled |
| Fabric Capacity Metrics app | Does not support Private Link |
| OneLake Catalog - Govern tab | Not available when Private Link is activated |
| Private link REST APIs | Do not support tags |
| Eventstream: Custom Endpoint source/destination | Not supported |
| Eventstream: Eventhouse as destination (direct ingestion) | Not supported |
| Eventstream: Activator as destination | Not supported |
| Eventhouse: OneLake ingestion | Not supported |
| Eventhouse: Shortcut creation | Not supported |
| Eventhouse: Pipeline connection | Not supported |
| Eventhouse: Queued ingestion | Not supported |
| Eventhouse: T-SQL queries | Not supported |
| Azure Events with Block Public Internet Access enabled | New event delivery blocked; existing configs stop delivering |
| Microsoft Purview Information Protection | Does not support Private Link (Sensitivity button grayed out in Desktop) |
| Most mirrored database types (see feature availability table) | Enter paused state when Block Public Internet Access is enabled |

### Browser URLs Still Required

Even with private links enabled, the client browser must be able to reach these URLs (typically via your VPN/split-tunnel config):

**Authentication:**
- `login.microsoftonline.com`
- `aadcdn.msauth.net`
- `msauth.net`
- `msftauth.net`
- `graph.microsoft.com`
- `login.live.com`

**Data Engineering / Data Science experiences:**
- `res.cdn.office.net`
- `aznbcdn.notebooks.azure.net`
- `pypi.org/*`
- `cdn.jsdelivr.net/npm/monaco-editor*`
- Local static endpoints for condaPackages

## Spark and Managed VNets

When the Azure Private Link tenant setting is enabled, **the first Spark job submitted in a workspace triggers creation of a managed VNet** for that workspace. This has important operational implications:

- Starter pools (pre-warmed shared VNet clusters) are **disabled** once the managed VNet is provisioned
- Spark jobs run on custom pools created on-demand within the dedicated managed VNet
- Cold start time increases to **3–5 minutes** for new Spark sessions
- Workspace migration across capacities in different regions is **not supported** after a managed VNet is allocated
- Spark jobs **do not work** for tenants whose home region doesn't support Fabric Data Engineering, even if they use capacities from other regions

## Bandwidth and Latency Considerations

All traffic routes through the private endpoint when private links are active. This includes:
- Data traffic (expected)
- Static assets — CSS, JavaScript, HTML files used by the Fabric portal

Users geographically far from the private endpoint location will experience **increased load times** because static assets no longer load from the nearest CDN edge but from the endpoint's location.

> **Example**: Australian users with a US-based private endpoint will load all Fabric portal assets from the US, adding significant latency.

## Costs

- [Azure Private Link pricing](https://azure.microsoft.com/pricing/details/private-link/) applies
- ExpressRoute bandwidth costs may increase if private connectivity is added from on-premises networks
- Custom Spark pool cold-start delays impact productivity and may require capacity right-sizing

## Disabling Private Link

Before disabling the Private Link setting:
1. Delete all private endpoints you created
2. Delete the corresponding private DNS zones
3. Then disable the setting in the Admin portal
4. Allow up to 15 minutes for changes to propagate

If private endpoints exist in your VNet but Private Link is disabled, connections from that VNet may fail. Schedule disablement during non-business hours.
