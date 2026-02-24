# Security in Microsoft Fabric — Overview

> **Source:** [Microsoft Learn — Security in Microsoft Fabric](https://learn.microsoft.com/en-us/fabric/security/security-overview)
> Last reviewed: 2026-02-24

## What Fabric Security Is

Microsoft Fabric is a SaaS platform. Unlike PaaS services where you configure security per resource, Fabric handles much of the security baseline for you: encryption at rest, TLS in transit, and Microsoft Entra ID authentication are on by default with no setup required.

Fabric security is described by Microsoft as:

- **Always on** — every interaction is encrypted and authenticated via Entra ID by default
- **Compliant** — data sovereignty via multi-geo, wide compliance standard support
- **Governable** — data lineage, sensitivity labels, DLP, Purview integration
- **Configurable** — layered controls you add on top of the baseline
- **Evolving** — new features and controls added regularly (many still in Preview)

## Authentication Baseline

Every request to Fabric is authenticated with **Microsoft Entra ID**. This includes browser sessions, Power BI Desktop, SSMS connections, API calls, and service-to-service communication. Fabric does not support legacy authentication methods (account keys, SQL authentication with username/password at the Fabric layer).

## Network Security — Two Directions

Fabric network security divides into two concerns:

### Inbound (traffic coming INTO Fabric)

Controlling where connections to Fabric originate from.

- By default, Fabric is accessible from the public internet (authenticated via Entra ID)
- Organizations add restrictions via Conditional Access, Private Links, or IP Firewall rules
- See [01-inbound/README.md](../01-inbound/README.md) for the decision guide

### Outbound (traffic FROM Fabric to data sources)

Controlling how Fabric workloads connect to external data sources.

- By default, Fabric can connect to public endpoints; firewall-protected sources require explicit configuration
- Options: Managed Private Endpoints, Trusted Workspace Access, Data Gateways, Service Tags
- See [02-outbound/README.md](../02-outbound/README.md) for the decision guide

## Internal Traffic

Traffic between Fabric experiences (e.g., a Power BI report loading data from OneLake) travels over the **Microsoft backbone network**, not the public internet. This is an important distinction from PaaS architectures where you typically need to connect services over private networks yourself.

## Data Security

- **At rest**: All Fabric data stores encrypted using Microsoft-managed keys by default. Customer-managed keys (CMK) available at the workspace level via Azure Key Vault.
- **In transit**: TLS 1.2 minimum enforced for all inbound communication; TLS 1.3 negotiated where possible. Outbound to customer infrastructure requires TLS 1.2+.
- **OneLake**: Single unified data lake backing all Fabric items. Workspace is the primary security boundary.

## Access Control

- Workspaces are the primary access control boundary
- Four workspace roles: Admin, Member, Contributor, Viewer
- Viewer role does NOT have OneLake access (cannot read underlying files)
- Item-level sharing allows granting access below workspace level
- Row-level security (RLS), column-level security (CLS), and object-level security (OLS) available for further restriction

## Key Capabilities Table

| Capability | Description | Doc |
|-----------|-------------|-----|
| Conditional Access | Identity-based access control via Entra ID | [conditional-access.md](../01-inbound/conditional-access.md) |
| Tenant Private Link | Network isolation for entire tenant | [private-links-tenant.md](../01-inbound/private-links-tenant.md) |
| Workspace Private Link | Network isolation per workspace | [private-links-workspace.md](../01-inbound/private-links-workspace.md) |
| Workspace IP Firewall | IP allowlist per workspace | [ip-firewall.md](../01-inbound/ip-firewall.md) |
| Managed VNets | Dedicated VNet for Spark workloads | [managed-vnets.md](../02-outbound/managed-vnets.md) |
| Managed Private Endpoints | Secure outbound to Azure PaaS | [managed-private-endpoints.md](../02-outbound/managed-private-endpoints.md) |
| Trusted Workspace Access | Secure access to firewall-enabled ADLS Gen2 | [trusted-workspace-access.md](../02-outbound/trusted-workspace-access.md) |
| Outbound Access Protection | Block all outbound + allowlist | [outbound-access-protection.md](../02-outbound/outbound-access-protection.md) |
| Service Tags | Azure NSG/Firewall tags for Fabric IPs | [service-tags.md](../02-outbound/service-tags.md) |
| Workspace Identity | Auto-managed service principal per workspace | [workspace-identity.md](../03-identity/workspace-identity.md) |
| Customer Managed Keys | Bring-your-own encryption key | [customer-managed-keys.md](../04-data-security/customer-managed-keys.md) |
| Customer Lockbox | Control Microsoft engineer access | *(see Microsoft docs)* |
| Row-level Security | Data-level access filtering | [row-level-security.md](../04-data-security/row-level-security.md) |
