# Workspace-Level Private Links

> **Sources:**
> - [Overview of workspace-level private links](https://learn.microsoft.com/en-us/fabric/security/security-workspace-level-private-links-overview)
> - [Supported scenarios and limitations](https://learn.microsoft.com/en-us/fabric/security/security-workspace-level-private-links-support)
> - [Blog: Fabric Workspace-Level Private Link Preview](https://blog.fabric.microsoft.com/en-US/blog/fabric-workspace-level-private-link-preview/) — Aug 20, 2025
> - [Blog: Announcing GA of Workspace-Level Private Link](https://blog.fabric.microsoft.com/en-us/blog/announcing-general-availability-of-workspace-level-private-link-in-microsoft-fabric) — Oct 1, 2025
> Last reviewed: 2026-02-24

## Feature Timeline

| Date | Event |
|------|-------|
| May 2024 | Tenant-level Private Link goes GA |
| Aug 20, 2025 | Workspace-level Private Link enters **Preview** — API management only; Fabric portal configuration not supported |
| Oct 1, 2025 | Workspace-level Private Link goes **GA** — Fabric portal management added |

> **Why this matters for community content**: Any blog post, video, or forum post from before October 2025 saying workspace-level private links can only be configured via API is describing the Preview behavior. As of GA, the Fabric portal supports the full configuration workflow.

## What It Is

Workspace-level Private Links allow you to network-isolate **individual workspaces** rather than the entire tenant. Each workspace gets its own Azure Private Link service, and a workspace-specific FQDN is used to route traffic privately. This enables a mixed model: some workspaces are fully isolated, others remain publicly accessible.

## Key Differences from Tenant-Level Private Links

| | Tenant-Level | Workspace-Level |
|--|-------------|----------------|
| Scope | All workspaces | Individual workspaces |
| FQDN | Tenant-specific | Workspace-specific |
| Fabric portal access when public blocked | Yes (via tenant PL network) | No (portal requires tenant PL; workspace PL only covers API) |
| SKU requirement | Any Fabric SKU | F SKU only (not P SKU, not Trial) |
| Mixed public/private | No | Yes |
| Admin required | Fabric admin + Azure | Workspace admin + Azure (tenant must enable) |

## Workspace FQDN Format

When using workspace-level private links, you must use the workspace-specific FQDN rather than the generic Fabric endpoints:

```
https://{workspaceid}.z{xy}.w.api.fabric.microsoft.com
https://{workspaceid}.z{xy}.c.fabric.microsoft.com
https://{workspaceid}.z{xy}.onelake.fabric.microsoft.com
https://{workspaceid}.z{xy}.dfs.fabric.microsoft.com
https://{workspaceid}.z{xy}.blob.fabric.microsoft.com
```

For Warehouse TDS connections:
```
https://{GUID}-{GUID}.z{xy}.datawarehouse.fabric.microsoft.com
```

Where:
- `{workspaceid}` = workspace object ID **without dashes**
- `{xy}` = first two characters of the workspace object ID

> **Critical**: If the FQDN is formatted incorrectly, it will not resolve to the intended private IP and the connection will fail. Get the workspace object ID from the URL bar when viewing the workspace in the Fabric portal.

## FQDN Resolution Behavior

| Environment | Resolution |
|-------------|------------|
| No private link configured | Public IP address |
| Tenant-level private link configured | Private IP (tenant-level config) |
| Workspace-level private link configured | Private IP (workspace-level config) |
| Both tenant and workspace private link (same VNet) | Private IP (workspace-level takes precedence) |
| Workspace-level private link for a **different** workspace | Public IP (if fallback enabled) or fails |

## Supported Item Types

The following item types support workspace-level private links:

- Lakehouse, SQL Endpoint, Shortcut
- Direct connection via OneLake endpoint
- Notebook, Spark Job Definition, Environment
- Machine learning experiment, machine learning model
- Pipeline, Copy Job, Mounted Data Factory
- Warehouse
- Dataflows Gen2 (CI/CD)
- Variable library
- Mirrored database (limited — see below)
- Eventstream (limited)
- Eventhouse (limited)

### Unsupported Item Types

- **Deployment Pipelines** — workspaces assigned to a deployment pipeline cannot restrict public access; deployment pipelines cannot connect to workspaces with access restricted
- **Default Semantic Models** — existing lakehouses, warehouses, and mirrored databases create a default semantic model that does not support workspace private links, blocking the ability to restrict public access
  - **Workaround**: Configure the workspace to block public access first, then create the lakehouse/warehouse/mirrored database. The default semantic model created after the restriction is in place will be compatible.

> **Important**: If a workspace contains any unsupported item types, you cannot restrict public inbound access for that workspace, even if workspace-level private link is configured.

## Private Link Service Architecture

- Each workspace has **exactly one** private link service (1:1 relationship)
- A single private link service can have **multiple private endpoints** (up to 100 per workspace)
- Multiple VNets can connect to the same workspace using separate private endpoints
- One VNet can connect to multiple workspaces using separate private endpoints

## Tenant Limits

- Max workspace private link services per tenant: **500**
- Max private endpoints per workspace: **100**
- Max workspace private link services created per minute: **10**
- Workspaces with active private link services cannot be deleted

## Restricting Public Access

Public access restriction is independent of private link setup:
- You can have a private link set up but still allow public access (useful for testing — not recommended for production)
- You can restrict public access without a private link — the workspace becomes inaccessible from all networks
- When public access is restricted, only connections through the associated private endpoint work

## Combining Inbound + Outbound Protection

The Fabric portal UI **does not currently support** enabling both workspace-level private links (inbound) and outbound access protection simultaneously. To configure both:

- Use the [Workspaces - Set Network Communication Policy API](https://learn.microsoft.com/en-us/rest/api/fabric/core/workspaces/set-network-communication-policy)

## Cross-Workspace Access

To connect from a workspace with outbound access protection to another workspace that has restricted inbound access:
1. Create a managed private endpoint in the source workspace pointing to the target workspace's private link service
2. Get approval from the target workspace private link service owner in Azure

Shortcuts and other cross-workspace data access scenarios require this setup when both workspaces have network restrictions.

## Known Limitations

| Limitation | Detail |
|-----------|--------|
| OneLake Security | Not currently supported when workspace-level private link is enabled |
| Workspace monitoring | Not currently supported with workspace-level private link |
| OneLake Catalog - Govern tab | Not available when Private Link is activated |
| Deployment pipelines | Cannot be used with restricted workspaces |
| Item sharing | Shared item links stop working for restricted workspaces |
| Spark friendly names | Workspace-level private links don't work with Spark friendly names |
| Shortcut transforms | Not supported in restricted workspaces |
| SSMS | Supported for warehouse connections |
| Monitoring hub Level 2 deeplinks | May not work as expected; navigate from L1 page instead |
| Eventstream: Custom Endpoint source/destination | Not supported |
| Eventstream: Eventhouse direct ingestion destination | Not supported |
| Eventstream: Activator destination | Not supported |
| Eventhouse: Eventstream consumption | Not supported |
| Eventhouse: SQL Server TDS endpoints | Not supported |
| Mirroring (most types) | Paused state when public access is restricted; only open mirroring, Cosmos DB, Azure SQL MI, SQL Server 2025 supported |
| Power Platform Dataflow Connector between workspace dataflows | Cannot connect when public access is denied |
| Dataflow Gen2 VNet gateway | Must reside in the same VNet as the workspace-level private link endpoint |
| Workspace staging (Copy activity) | Not supported for Fabric Data Warehouse, Snowflake, or Teradata connectors — use external staging |
| Copy to Eventhouse | Not supported |

## Error Reference

**RequestDeniedByInboundPolicy**
```json
{
  "errorCode": "RequestDeniedByInboundPolicy",
  "message": "Request is denied due to inbound communication policy"
}
```
Cause: Request originates from a network not allowed by the workspace's communication policy.
Fix: Verify you are on the correct network; ensure you are using the workspace FQDN (not the generic endpoint).

**InboundRestrictionNotEligible**
```json
{
  "errorCode": "InboundRestrictionNotEligible",
  "message": "This workspace contains items that do not comply with requested policy"
}
```
Cause: Workspace contains unsupported item types (e.g., default semantic models, deployment pipelines).
Fix: Remove unsupported items first, or use the workaround for default semantic models (configure restriction before creating the item).

## REST API Coverage

APIs under `v1/workspaces/{workspaceId}` support workspace private links.

Admin APIs (`admin/workspaces/{workspaceId}`) are **not** covered — they remain accessible from public networks even when public access is restricted for the workspace. Tenant-level settings govern admin APIs.

The network communication policy API is similarly accessible from public networks even with workspace restrictions in place.
