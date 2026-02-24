# Workspace Identity

> **Source:** [Workspace identity — Microsoft Fabric](https://learn.microsoft.com/en-us/fabric/security/workspace-identity)
> Last reviewed: 2026-02-24

## What It Is

A workspace identity is an automatically managed service principal that Fabric associates with a specific workspace. It is created and managed entirely within Fabric — you do not need to create or rotate credentials manually.

When a workspace identity is created:
- Fabric creates a service principal in Microsoft Entra ID
- An accompanying app registration is also created
- Fabric manages all credentials automatically (no credential leaks, no downtime from credential rotation)
- The identity name always matches the workspace name

## Primary Uses

1. **Trusted workspace access** — enables Fabric workloads (shortcuts, pipelines, semantic models) to access firewall-enabled ADLS Gen2 accounts using the workspace as an identity (see [trusted-workspace-access.md](../02-outbound/trusted-workspace-access.md))
2. **Authentication** — surfaces as an authentication option in OneLake shortcuts, pipelines, semantic models, and Dataflows Gen2 when connecting to data sources that support Entra ID authentication

## Creating a Workspace Identity

- Requires **workspace admin** role
- Cannot be created in **My Workspace**
- Available in workspace settings → **Workspace identity** tab → **+ Workspace identity**

**Tenant limit**: 10,000 workspace identities by default (configurable in tenant settings)

## Access Control and Security

By default, the workspace identity is **not** granted any workspace role when created.

Users who can configure the workspace identity for authentication in connections: admin, member, or contributor role in the workspace.

> **Security warning from Microsoft**: The workspace identity is an automatically managed service principal. Any user given access to the identity is allowed to assume it. Access should be carefully managed and monitored.

Application Administrators (or higher) in Entra ID can view, modify, and delete the associated service principal and app registration in Azure — but this is strongly discouraged as it will break Fabric items relying on the identity.

## Lifecycle Considerations

| Event | Effect |
|-------|--------|
| Workspace deleted | Identity is deleted |
| Workspace restored after deletion | Identity is NOT restored — must create a new one |
| Workspace renamed | Identity name updates to match; the underlying Entra app/service principal remains the same |
| Workspace migrated to non-F SKU capacity | Identity is not disabled/deleted, but trusted workspace access stops working after ~1 hour |

## Administration

### In Fabric Admin Portal

Fabric admins can view and manage all workspace identities in the tenant via the **Fabric identities** tab in the admin portal. They can view details and delete identities.

### In Microsoft Purview (Audit Logs)

Audit events are generated for:
- Created Fabric Identity for Workspace
- Retrieved Fabric Identity for Workspace
- Deleted Fabric Identity for Workspace
- Retrieved Fabric Identity Token for Workspace

### In Azure Portal

The service principal appears under **Enterprise Applications** in Azure. Do not modify it there — changes will cause Fabric items to break and may be automatically reverted.

The app registration appears under **App Registrations** in Azure. Same guidance — do not modify.

## Conditional Access Interaction

If your organization has a Conditional Access policy for workload identities that **includes all service principals**, trusted workspace access will not work. You must explicitly exclude specific Fabric workspace identities from that policy.

## Limitations

- Cannot be created in My Workspace
- Default tenant limit: 10,000 identities (configurable)
- If workspace identity state shows as **Failed**: wait 1 hour, delete it, wait 5 minutes, create again
- After identity deletion, cannot be restored — must create new

## Relationship to Other Features

| Feature | Requires Workspace Identity? |
|---------|------------------------------|
| Trusted Workspace Access (ADLS Gen2) | Yes |
| OneLake shortcuts with trusted access | Yes |
| Authentication in pipelines (to ADLS Gen2) | Yes |
| Authentication in semantic models (import mode, ADLS Gen2) | Yes |
| Managed Private Endpoints | No |
| Workspace Private Links | No |
