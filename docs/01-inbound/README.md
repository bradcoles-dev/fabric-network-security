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

## Tenant Admin Enablement Required for Workspace Controls

Workspace-level inbound rules (private links, IP firewall) are **disabled by default** at the workspace level. A Fabric tenant administrator must enable the **"Configure workspace-level inbound network rules"** tenant setting before workspace admins can apply workspace-level restrictions.

---

## Documents in This Section

- [conditional-access.md](conditional-access.md) — Entra Conditional Access for Fabric
- [private-links-tenant.md](private-links-tenant.md) — Tenant-level Private Link: overview, setup, limitations
- [private-links-workspace.md](private-links-workspace.md) — Workspace-level Private Link: overview, supported items, limitations
- [ip-firewall.md](ip-firewall.md) — Workspace IP Firewall rules
