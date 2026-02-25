# Client Project — Discovery Summary

> For full detail see [detail.md](detail.md)
> Last updated: 2026-02-25

## What We're Building

Medallion lakehouse (Landed → Bronze → Silver → Gold) on Microsoft Fabric. On-premises SQL ingestion via OPDG, Azure Blob and S3 shortcuts, Genesys PureCloud API. Spark notebooks for transformation, Power BI for reporting. DEV / UAT / PROD as separate workspaces, each on its own F SKU capacity. Microsoft Purview in scope.

---

## Inbound: Network Isolation Decision — Not Yet Made

**The RFP contains no Private Link, network isolation, or VNet requirement.** The RFP explicitly requires [Conditional Access + MFA](../docs/01-inbound/conditional-access.md) — which, with the client's existing E5 licensing, is the contractual baseline. Private Link is only required if a compliance assessment drives it: specifically, if **PCI-DSS scope includes cardholder data flowing through Fabric** (network controls become mandatory), or if the **APRA CPS 234 risk assessment** identifies network isolation as a required control. Until that assessment is done, CA + MFA is the default position.

Six options are available if network-layer isolation is subsequently required:

| Option | Isolation Layer | OPDG Works | Capacity Metrics App | MIP Labels | Operational Overhead | Viable? |
|--------|----------------|-----------|----------------------|------------|----------------------|---------|
| No isolation | None | ✓ | ✓ | ✓ | Low | Yes (DEV) |
| [Conditional Access](../docs/01-inbound/conditional-access.md) only (E5 — already licensed) | Authentication | ✓ | ✓ | ✓ | Low | **Yes — contractual baseline; default position** |
| [Workspace IP Firewall](../docs/01-inbound/ip-firewall.md) only | Network (IP allowlist) | ✓ | ✓ | ✓ | Low | Yes — lightweight additive control |
| [Workspace-level Private Link](../docs/01-inbound/private-links-workspace.md) | Network (private routing) | ✓ | ✓ (unconfirmed) | ✓ (unconfirmed) | Medium | Only if PCI-DSS or APRA risk assessment requires it |
| Tenant-level PL (no Block Public) | Network (private routing) | Possibly | ✓ | ✓ | Medium | Risky — no advantage over workspace-level |
| Tenant-level PL + Block Public Internet | Network (full isolation) | ✗ | ✗ | ✗ | High | **No** |

> **The CA question to put to Cyber Security Policy**: Is the compliance driver *identity-based* (only approved users on approved devices can access Fabric) or *network-layer* (Fabric traffic must never traverse the public internet)? CA satisfies the first. Only Private Link satisfies the second. Many clients conflate the two. With ZScaler egress IPs as a named location, CA effectively requires users to be on corporate ZScaler — turning ZScaler from a blocker into the control mechanism.

---

## Outbound: Data Source Connectivity — Also Not Decided

The outbound question is separate from inbound: *what networks do Fabric workloads use to reach data sources, and should outbound connections be restricted to pre-approved destinations?* Two sub-decisions:

**1. Private connectivity per data source** — should Fabric reach sources over private networks rather than the public internet?

| Data Source | Default Path | Private Option | Mechanism | Status |
|-------------|-------------|----------------|-----------|--------|
| MS SQL Server | OPDG relay — on-prem leg is private; Fabric↔OPDG leg uses Azure Service Bus (public internet, encrypted) | Full private path via ExpressRoute/VPN routing of OPDG machine's Azure traffic | [OPDG](../docs/02-outbound/data-gateways.md) | In scope; full private path requires ExpressRoute/VPN (open question) |
| Sybase ASE | Same as above (ODBC driver required) | Full private path via ExpressRoute/VPN routing of OPDG machine's Azure traffic | [OPDG](../docs/02-outbound/data-gateways.md) | In scope; drives inbound decision; full private path requires ExpressRoute/VPN (open question) |
| Azure Blob Storage | Public internet | Yes | [MPE](../docs/02-outbound/managed-private-endpoints.md) (Spark) / [TWA](../docs/02-outbound/trusted-workspace-access.md) (auth only) | Open — needs decision |
| Amazon S3 | Public internet | Yes (via OPDG) | [OPDG-backed OneLake shortcut](../docs/02-outbound/data-gateways.md) | Conditional — OPDG must reach S3; only viable under workspace-level PL (not tenant-level PL + block) |
| Genesys PureCloud | Public internet (SaaS API) | No | N/A — public SaaS | Stays public; allowlist if [OAP](../docs/02-outbound/outbound-access-protection.md) enabled |
| Azure SQL (control DB) | TBD — not yet provisioned | Yes (if network-restricted) | [MPE](../docs/02-outbound/managed-private-endpoints.md) (Notebooks) / [VNet gateway](../docs/02-outbound/data-gateways.md) (Pipelines) | Open — depends on DB network config |

**2. Outbound Access Protection (OAP)** — should Fabric workloads be restricted to connecting only to pre-approved destinations (data exfiltration prevention)?

[OAP](../docs/02-outbound/outbound-access-protection.md) is the outbound complement to workspace-level Private Link. It blocks all outbound connections by default and requires explicit allowlisting per endpoint. Spark GA Oct 2025; Pipelines support announced Oct 2025. **Not yet scoped for this client.** Key constraints if it comes into scope:
- F SKU required
- Cannot configure both inbound Private Link and OAP via the Fabric portal — must use the REST API
- Incompatible with external data sharing (cross-tenant)

---

## Key Concerns

### 1. Sybase ASE rules out tenant-level Private Link
Sybase ASE requires OPDG with an ODBC driver. [OPDG registration fails](../docs/02-outbound/data-gateways.md) under [tenant-level Private Link with Block Public Internet Access](../docs/01-inbound/private-links-tenant.md) enabled — a hard blocker. This single data source effectively decides the isolation model: **workspace-level PL only, no tenant-level PL**.

### 2. ZScaler must be resolved before any Private Link testing
ZScaler sits between all users and Fabric. It intercepts DNS and may perform SSL inspection. If Fabric FQDNs are not in ZScaler's bypass list, Private Link silently fails. This is a dependency on the client's Cyber Security Operations team and their change management process — it must be resolved before any network testing begins.

### 3. Capacity Metrics App — needs confirmation
The app is [documented as incompatible with tenant-level Private Link](../docs/01-inbound/private-links-tenant.md). Compatibility with workspace-level PL is assumed but **unconfirmed**. Without the Capacity Metrics App, there is no meaningful capacity management or cost visibility in PROD. If workspace-level PL also breaks it, the isolation model needs to be reconsidered.

### 4. Purview sensitivity labels — unconfirmed with workspace-level PL
[MIP sensitivity labels are confirmed broken with tenant-level PL](../docs/01-inbound/private-links-tenant.md). Impact of workspace-level PL is **undocumented**. If labelling is mandatory, this needs a proof-of-concept test before architecture is locked. Purview scanning of restricted workspaces is also unconfirmed.

### 5. Amazon S3 — private connectivity possible via OPDG-backed shortcut
[OneLake shortcuts to S3](https://learn.microsoft.com/en-us/fabric/onelake/onelake-shortcuts) can be backed by an OPDG, routing privately through the gateway rather than over the public internet — provided the OPDG machine has network access to the S3 endpoint ([MS docs, updated 2026-02-20](https://learn.microsoft.com/en-us/fabric/onelake/create-on-premises-shortcut)). Since OPDG is already required for Sybase ASE, this may add no additional infrastructure. If the OPDG cannot reach S3, staging to Azure Blob first is the fallback. **Requires confirmation of OPDG network topology and policy scope from Cyber Security Policy.**

### 6. Deployment Pipelines incompatible with restricted workspaces
[Workspaces with public access restricted cannot use Fabric Deployment Pipelines](../docs/01-inbound/private-links-workspace.md) for DEV → UAT → PROD promotion. **[Git-based CI/CD](https://learn.microsoft.com/en-us/fabric/cicd/git-integration/intro-to-git-integration) (Azure DevOps or GitHub) is required for any restricted workspace.** This is the recommended enterprise pattern regardless — but needs to be agreed with the Head of Data early, not discovered during build.

### 7. Workspace provisioning sequencing
If workspace-level PL with public access restriction is used, the restriction must be configured **before** any Lakehouse or Warehouse is created in that workspace. [Creating these items first auto-generates default semantic models that are incompatible with the restriction](../docs/01-inbound/private-links-workspace.md), blocking it permanently. This must be built into the provisioning runbook from day one.

### 8. Scale overhead
10–20 workspaces × 3 environments = potentially 30–60+ private link setups (private endpoint, DNS entries per workspace). A tiered approach is recommended: workspace-level PL for sensitive PROD workspaces only, IP Firewall elsewhere. Applying PL universally is disproportionate and operationally expensive.

### 9. Outbound Access Protection not yet scoped
[OAP](../docs/02-outbound/outbound-access-protection.md) is not in the current conversation. If the client's Cyber Security Policy team also wants to restrict what Fabric workloads can connect to outbound — a data exfiltration concern, not just an access control concern — this adds scope. If OAP lands alongside workspace-level Private Link, both must be configured via the REST API (portal cannot do both simultaneously). This question should be raised in discovery rather than discovered during build.

---

## Questions to Resolve in Discovery

| Question | Owner |
|----------|-------|
| Does cardholder data (payment card numbers, CVVs, expiry dates) flow through Fabric? PCI-DSS network controls only apply if yes. | Head of Cyber Security Policy + Head of Data |
| Is the isolation requirement identity-based (who can access) or network-layer (traffic must not traverse public internet)? CA satisfies the former; only Private Link satisfies the latter. | Head of Cyber Security Policy |
| What specific compliance control in APRA CPS 234 or PCI-DSS requires network isolation beyond CA + MFA? | Head of Cyber Security Policy |
| Are existing Conditional Access policies scoped to Power BI Service only? If so, they must be expanded to all five Fabric-dependent services before relying on CA as a control. | Head of Cyber Security Operations |
| Is Purview sensitivity labelling mandatory? | Head of Cyber Security Policy + Head of Data |
| Does "no public internet" apply to outbound Fabric connections (e.g., S3)? | Head of Cyber Security Policy |
| Is the Capacity Metrics App required in PROD? | Head of Cyber Security Operations |
| Are Deployment Pipelines planned for environment promotion? | Head of Data |
| Does VPN/ExpressRoute exist from on-premises to Azure? | Head of Architecture |
| Can ZScaler bypass rules be added for Fabric FQDNs? | Head of Cyber Security Operations |
| Confirm Capacity Metrics App works under workspace-level PL | Internal — test required |
| Confirm Purview labelling works under workspace-level PL | Internal — test required |
| Is there a requirement to restrict which external endpoints Fabric workloads can connect to outbound (data exfiltration prevention)? | Head of Cyber Security Policy |
| Should Azure Blob Storage be accessed via managed private endpoint (Spark) or trusted workspace access? | Head of Architecture |
| Will the Azure SQL control DB be network-restricted? If so: MPE for Notebooks, VNet gateway for Pipelines | Head of Architecture |
