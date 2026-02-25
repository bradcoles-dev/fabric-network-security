# Client Project — Discovery Summary

> For full detail see [detail.md](detail.md)
> Last updated: 2026-02-25

## What We're Building

Medallion lakehouse (Landed → Bronze → Silver → Gold) on Microsoft Fabric. On-premises SQL ingestion via OPDG, Azure Blob and S3 shortcuts, Genesys PureCloud API. Spark notebooks for transformation, Power BI for reporting. DEV / UAT / PROD as separate workspaces, each on its own F SKU capacity. Microsoft Purview in scope.

---

## Network Isolation Decision — Not Yet Made

**This is the single decision that drives everything else.** Five options exist, from no isolation to full tenant-level Private Link with all public internet blocked. The right answer for this client is likely **[workspace-level Private Link](../docs/01-inbound/private-links-workspace.md) on sensitive PROD workspaces only**, combined with [Workspace IP Firewall](../docs/01-inbound/ip-firewall.md) elsewhere. Full tenant-level isolation is not viable (see key concerns below).

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

### 5. Amazon S3 has no private connectivity path from Fabric
OneLake S3 shortcuts use the public internet. If "no public internet" is a hard policy requirement covering all Fabric connections — not just inbound — S3 data must be staged to Azure Blob first via an external process. **Requires policy clarification from Cyber Security Policy.**

### 6. Deployment Pipelines incompatible with restricted workspaces
[Workspaces with public access restricted cannot use Fabric Deployment Pipelines](../docs/01-inbound/private-links-workspace.md) for DEV → UAT → PROD promotion. **[Git-based CI/CD](https://learn.microsoft.com/en-us/fabric/cicd/git-integration/intro-to-git-integration) (Azure DevOps or GitHub) is required for any restricted workspace.** This is the recommended enterprise pattern regardless — but needs to be agreed with the Head of Data early, not discovered during build.

### 7. Workspace provisioning sequencing
If workspace-level PL with public access restriction is used, the restriction must be configured **before** any Lakehouse or Warehouse is created in that workspace. [Creating these items first auto-generates default semantic models that are incompatible with the restriction](../docs/01-inbound/private-links-workspace.md), blocking it permanently. This must be built into the provisioning runbook from day one.

### 8. Scale overhead
10–20 workspaces × 3 environments = potentially 30–60+ private link setups (private endpoint, DNS entries per workspace). A tiered approach is recommended: workspace-level PL for sensitive PROD workspaces only, IP Firewall elsewhere. Applying PL universally is disproportionate and operationally expensive.

---

## Questions to Resolve in Discovery

| Question | Owner |
|----------|-------|
| What specific compliance control requires network isolation? | Head of Cyber Security Policy |
| Is Purview sensitivity labelling mandatory? | Head of Cyber Security Policy + Head of Data |
| Does "no public internet" apply to outbound Fabric connections (e.g., S3)? | Head of Cyber Security Policy |
| Is the Capacity Metrics App required in PROD? | Head of Cyber Security Operations |
| Are Deployment Pipelines planned for environment promotion? | Head of Data |
| Does VPN/ExpressRoute exist from on-premises to Azure? | Head of Architecture |
| Can ZScaler bypass rules be added for Fabric FQDNs? | Head of Cyber Security Operations |
| Confirm Capacity Metrics App works under workspace-level PL | Internal — test required |
| Confirm Purview labelling works under workspace-level PL | Internal — test required |
