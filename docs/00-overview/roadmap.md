# Fabric Network Security — Roadmap Items

> **Source:** [Microsoft Fabric Roadmap](https://roadmap.fabric.microsoft.com/?product=administration%2Cgovernanceandsecurity)
> Last reviewed: 2026-02-24

Items below are scoped to network security and governance features only. All items are from the official Microsoft Fabric public roadmap and are subject to change.

> **Q1 2026 note**: Items listed as Q1 2026 are either already released or will release by end of March 2026. As of the last review date (2026-02-24), their exact release status has not been independently confirmed — verify against current documentation before relying on availability.

## Planned / Upcoming

| Feature | Release Target | Release Type | Relevant Doc |
|---------|---------------|--------------|--------------|
| Workspace-level Private Link for SQL Database | Q1 2026 | GA | [private-links-workspace.md](../01-inbound/private-links-workspace.md) |
| Workspace Public IP Firewall | Q1 2026 | GA | [ip-firewall.md](../01-inbound/ip-firewall.md) |

## Item Details

### Workspace-level Private Link for SQL Database

**Roadmap description:** Private link allows customers to connect to their SQL DBs via a private endpoint from the customer's virtual network. Workspace-level private links will give customers the flexibility to use private link connection to specific workspace(s) which have high security requirements and contain sensitive data while their other workspaces can be open to public.

**What this means**: SQL Database in Fabric is not currently supported for workspace-level private links. This item adds it. See the unsupported items list in [private-links-workspace.md](../01-inbound/private-links-workspace.md) — currently, SQL Database is absent from the supported item types list.

**Release**: Q1 2026, GA

---

### Workspace Public IP Firewall

**Roadmap description:** Public IP firewall rules in Fabric provide a way to control workspace access based on the incoming request's IP address. Useful for organizations that want to allow access to Fabric workspaces over public networks while maintaining controlled, secure entry points.

**What this means**: The workspace IP firewall feature (documented in [ip-firewall.md](../01-inbound/ip-firewall.md)) is reaching GA. This confirms the feature was previously in preview. The GA release implies official support commitment, SLA eligibility, and removal of preview caveats.

**Release**: Q1 2026, GA
