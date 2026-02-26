# Inbound Network Security — Decision Guide

> **Sources:**
> - [About inbound access protection in Fabric](https://learn.microsoft.com/en-us/fabric/security/security-inbound-overview)
> - [Protect inbound traffic to Fabric](https://learn.microsoft.com/en-us/fabric/security/protect-inbound-traffic)
> Last reviewed: 2026-02-24

Inbound security controls where connections to Fabric originate from. Fabric provides controls at two scopes: **tenant-level** and **workspace-level**.

---

## Inbound Controls at a Glance

| Feature | Scope | Mechanism | Blocks Public Internet? | Requires Azure VNet? | Admin Level |
|---------|-------|-----------|------------------------|---------------------|-------------|
| Conditional Access | Tenant | Identity/policy | No (restricts, not blocks) | No | Entra Admin |
| Tenant Private Link | Tenant | Azure Private Link | Yes (optional) | Yes | Fabric Admin |
| Workspace Private Link | Workspace | Azure Private Link | Yes (optional) | Yes | Workspace Admin (tenant must enable) |
| Workspace IP Firewall | Workspace | IP allowlist | No (allows specific IPs) | No | Workspace Admin (tenant must enable) |

---

## Deciding Which Option to Use

### Option 1: Conditional Access (identity-based, no VNet required)

**Best for:** Organizations that want to restrict access based on user identity, location, device compliance, or MFA requirements without setting up Azure networking.

**How it works:** Microsoft Entra ID evaluates policies at authentication time. Access is granted or denied based on conditions like IP range, country, device state, or risk level.

**Key trade-off:** Conditional Access is broad — policies apply to Fabric and all downstream Azure services it depends on (Power BI Service, Azure Data Explorer, Azure SQL Database, Azure Storage, Azure Cosmos DB). You cannot target Fabric alone.

**When it's not enough:** Conditional Access does not enforce network-level isolation. Authenticated users can still access Fabric over the public internet. For organizations that need traffic never to traverse the public internet, Private Links are required.

### Option 2: Tenant-Level Private Link (network isolation for entire tenant)

**Best for:** Organizations that need to completely lock Fabric away from the public internet for all workspaces in the tenant.

**How it works:** A private endpoint is created in your Azure VNet. All Fabric traffic routes through this endpoint over Microsoft's backbone. Public internet access can optionally be blocked.

**Key trade-offs:**
- Requires Azure VNet infrastructure and DNS configuration
- Impacts many Fabric features — see [private-links-tenant.md](private-links-tenant.md) for the full limitations list
- All users must access Fabric through the private network (VPN, ExpressRoute, Bastion, etc.)
- Bandwidth: all traffic routes through the private endpoint, including static assets like CSS/JS — this increases latency for users geographically distant from the endpoint
- On-premises data gateways **do not work** with tenant Private Link; VNet data gateways do

### Option 3: Workspace-Level Private Link (per-workspace network isolation)

**Best for:** Organizations that need to isolate specific high-sensitivity workspaces while leaving others publicly accessible.

**How it works:** Each workspace gets its own Azure Private Link service. A workspace-specific FQDN is used to route traffic privately. You can restrict public access per workspace independently of the tenant setting.

**Key trade-offs:**
- Only supported on F SKU capacities (not P SKU or Trial)
- Unsupported item types (e.g., Deployment Pipelines, Default Semantic Models) prevent public access from being blocked in a workspace that contains them
- Cannot currently configure both inbound private links AND outbound access protection via the Fabric portal UI — must use the REST API
- 500 workspace private link services per tenant; 100 private endpoints per workspace

### Option 4: Workspace IP Firewall (IP-based allowlist)

**Best for:** Organizations that want lightweight IP-based access control without Private Link complexity.

**How it works:** You define a list of allowed public IP address ranges. Only traffic from those IPs can access the workspace.

**Key trade-offs:**
- Does not block traffic at the network level — still traverses public internet
- Only supports public IP addresses; RFC 1918 private addresses are not supported
- Max 256 rules per workspace
- Cannot add VM IPs that are on VNets with private endpoints as firewall rules
- IP firewall only controls inbound access — it has no effect on outbound connections

---

## How Tenant and Workspace Settings Interact

This is where it gets complex. The table below shows effective access behavior based on combinations of settings:

| Tenant Public Access | Network | Fabric Portal Accessible? | REST API Accessible? |
|---------------------|---------|--------------------------|---------------------|
| Allowed | Public internet | Yes | Yes (api.fabric.microsoft.com) |
| Allowed | Tenant private link network | Yes | Yes (tenant-specific FQDN) |
| Allowed | Workspace private link network | Yes | Yes (workspace-specific FQDN) |
| Restricted | Public internet | **No** | **No** |
| Restricted | Tenant private link network | Yes | Yes (tenant-specific FQDN or api.fabric.microsoft.com if client allows public) |
| Restricted | Workspace private link network | **No** (portal) | Yes (workspace-specific FQDN) |
| Restricted | Both tenant + workspace private link | Yes | Yes |

**Key insight**: When tenant public access is restricted, the Fabric portal (app.fabric.microsoft.com) is only accessible through the tenant private link network — not through workspace-level private links alone. Workspace-level private links give API access to the workspace but do not open the portal.

---

## Private Link Limitations Comparison

The table below maps every documented limitation across both Private Link types. **"Not documented"** means there is no official documentation confirming the feature works or is broken — do not assume support. **"Not applicable"** means the concept does not apply at that scope.

| Feature / Area | Tenant-Level PL | Workspace-Level PL |
|----------------|----------------|--------------------|
| **Setup** | | |
| SKU requirement | Any Fabric SKU | F SKU only — not P SKU, not Trial |
| Fabric portal access when public blocked | Accessible via tenant PL network | Not accessible — workspace PL covers API only; portal requires tenant-level PL |
| Admin APIs | Governed by tenant settings | Remain publicly accessible even with workspace restriction |
| Configure inbound + outbound via portal | Not supported simultaneously — must use REST API | Not supported simultaneously — must use REST API |
| Capacity limit | 450 capacities per tenant | 500 workspace private link services per tenant; 100 private endpoints per workspace |
| New capacities | Up to 24 hours to reflect in private DNS zone | Not applicable |
| Trial capacity | Not supported | Not supported |
| Tenant migration | Blocked while enabled | Not applicable |
| Workspace deletion | Not documented | Workspaces with active private link services cannot be deleted |
| SQL Database | Supported | Not yet supported — Q1 2026 roadmap |
| **Data Gateway** | | |
| On-premises data gateway (OPDG) | Fails when Block Public Internet Access is enabled; may work without it (officially unsupported) | Not affected — documented as Microsoft's recommended model for OPDG + PL coexistence |
| VNet Data Gateway download diagnostics | Does not work | Not documented |
| **Deployment** | | |
| Deployment Pipelines | Not documented | Cannot be used with restricted workspaces |
| Default Semantic Models | Not documented | Existing lakehouses/warehouses/mirrored databases generate incompatible default semantic models — must restrict workspace before creating these items |
| **Capacity Metrics App** | Does not support Private Link | Not documented |
| **Microsoft Purview / MIP** | | |
| Sensitivity labels (MIP) | Not supported — Sensitivity button grayed out in Power BI Desktop | Not documented |
| OneLake Catalog — Govern tab | Not available | Not available |
| **OneLake** | | |
| OneLake regional endpoints (direct calls) | Do not work via private link | Not documented |
| OneLake Security | Not documented | Not currently supported |
| Shortcut transforms | Not documented | Not supported in restricted workspaces |
| **Spark / Data Engineering** | | |
| Spark starter pools | Disabled once managed VNet is provisioned (triggered by tenant PL setting) | Not documented — managed VNet trigger is documented only for tenant PL setting |
| Spark cold start | Increases to 3–5 minutes after managed VNet provisioned | Not documented |
| Workspace migration across regions | Not supported after managed VNet is allocated | Not documented |
| Spark friendly names | Not documented | Do not work with workspace-level PL |
| **Bandwidth / Latency** | Static assets (CSS, JS) route through private endpoint — increased latency for geographically distant users | Not documented |
| **Monitoring** | | |
| Workspace monitoring | Not documented | Not currently supported |
| Monitoring hub Level 2 deeplinks | Not documented | May not work — navigate from Level 1 page instead |
| Power BI modern usage metrics | Partial only — Report Open events; no Page Views or performance data | Not documented |
| **Power BI** | | |
| Publish to Web | Not supported | Not documented |
| Email subscriptions | Not supported when Block Public Internet Access enabled | Not documented |
| Export report to PDF / PowerPoint | Not supported | Not documented |
| Copilot | Not supported | Not documented |
| Visual query in Warehouse | Does not work when Block Public Internet Access enabled | Not documented |
| Direct Lake (datamart/dataflow source, internet-blocked) | Connection fails | Not documented |
| **Item Sharing** | | |
| Shared item links | Not documented | Stop working for restricted workspaces |
| Power Platform Dataflow Connector between workspace dataflows | Not documented | Cannot connect when public access is denied |
| **Copy Activity** | | |
| Workspace staging (Warehouse, Snowflake, Teradata) | Not documented | Not supported — use external staging |
| Copy to Eventhouse | Not documented | Not supported |
| **Eventstream** | | |
| Custom Endpoint source/destination | Not supported | Not supported |
| Eventhouse as destination (direct ingestion) | Not supported | Not supported |
| Activator as destination | Not supported | Not supported |
| **Eventhouse** | | |
| OneLake ingestion | Not supported | Not documented |
| Eventstream consumption | Not documented | Not supported |
| Shortcut creation | Not supported | Not documented |
| Pipeline connection | Not supported | Not documented |
| Queued ingestion | Not supported | Not documented |
| T-SQL queries / SQL Server TDS endpoints | Not supported | Not supported |
| **Mirroring** | | |
| Most mirrored database types | Paused when Block Public Internet Access enabled — only open mirroring, Cosmos DB, Azure SQL MI, SQL Server 2025 supported | Paused when public access restricted — same supported types as tenant-level |
| **Azure Events** | | |
| Azure Events (Block Public Internet Access) | New event delivery blocked; existing configs stop delivering | Not documented |
| **Other** | | |
| Private link REST APIs | Do not support tags | Not documented |
| Dataflow Gen2 VNet gateway | Not documented | Must reside in same VNet as workspace-level PL endpoint |

---

## Tenant Admin Enablement Required for Workspace Controls

Workspace-level inbound rules (private links, IP firewall) are **disabled by default** at the workspace level. A Fabric tenant administrator must enable the **"Configure workspace-level inbound network rules"** tenant setting before workspace admins can apply workspace-level restrictions.

---

## Documents in This Section

- [conditional-access.md](conditional-access.md) — Entra Conditional Access for Fabric
- [private-links-tenant.md](private-links-tenant.md) — Tenant-level Private Link: overview, setup, limitations
- [private-links-workspace.md](private-links-workspace.md) — Workspace-level Private Link: overview, supported items, limitations
- [ip-firewall.md](ip-firewall.md) — Workspace IP Firewall rules
