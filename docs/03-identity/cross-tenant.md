# Cross-Tenant Communication

> **Sources:**
> - [About cross-tenant communication](https://learn.microsoft.com/en-us/fabric/security/security-cross-tenant-communication)
> - [Business to business (B2B guest users)](https://learn.microsoft.com/en-us/fabric/security/security-b2b)
> Last reviewed: 2026-02-24

## Two Different Scenarios

Cross-tenant in Fabric means two distinct things:

1. **B2B Guest Access** — external users from another Azure AD/Entra tenant accessing your Fabric workspace as guests (via Entra B2B invitation)
2. **Cross-Tenant Network Connectivity** — establishing private network connectivity between a Fabric workspace in one Azure tenant and resources/clients in another Azure tenant

---

## B2B Guest User Access

Sharing Fabric items with guest users works similarly to Power BI B2B, with one important restriction:

- In Fabric, you can only share items with guest users by sharing the **workspace** itself
- **Explicit item-level sharing with guest users is not supported**, except for: reports, dashboards, semantic models, and apps

For full details on guest user access configuration, refer to the [Power BI Microsoft Entra B2B documentation](https://learn.microsoft.com/en-us/fabric/enterprise/powerbi/service-admin-entra-b2b).

> **Enterprise note**: The limitation that guest users cannot be given item-level access (only workspace-level) is a meaningful restriction for organizations that want fine-grained external sharing. Plan workspace structures accordingly if external collaboration is a requirement.

---

## Cross-Tenant Private Network Connectivity

This scenario is for organizations that need to establish **private network access** from one Azure tenant to a Fabric workspace in a completely separate Azure tenant. This is typically relevant for:

- Managed service providers accessing client Fabric environments
- Subsidiary organizations with separate tenants that need to consume data from a parent tenant's Fabric workspace
- Data sharing between partner organizations without exposing data over the public internet

### Architecture

The setup uses workspace-level Private Link:

1. **Tenant 2** (the target): Has the Fabric workspace. Creates a Private Link service for that workspace using the `Microsoft.Fabric/privateLinkServicesForFabric` ARM resource type.
2. **Tenant 1** (the source): Creates a VNet, a VM, and a private endpoint that connects to the Private Link service resource in Tenant 2.

DNS in Tenant 1 must be configured to resolve the Fabric workspace FQDN to the private endpoint IP address using a private DNS zone: `privatelink.fabric.microsoft.com`.

### Key ARM Template Difference from Tenant-Level Private Link

For cross-tenant workspace private links, the ARM template uses a **different resource type** than the tenant-level private link:

```json
{
  "type": "Microsoft.Fabric/privateLinkServicesForFabric",
  "apiVersion": "2024-06-01",
  "properties": {
    "tenantId": "<tenant-id>",
    "workspaceId": "<workspace-id>"
  }
}
```

Note this is `Microsoft.Fabric/privateLinkServicesForFabric` (not `Microsoft.PowerBI/privateLinkServicesForPowerBI` which is used for tenant-level links).

### Prerequisites

- `Microsoft.Fabric` Resource Provider must be provisioned in **both** tenants
- Workspace in Tenant 2 must be on a Fabric capacity (F SKU)
- Private endpoint connection request must be approved by the Private Link service owner in Tenant 2

### Step Summary

1. Tenant 2: Create workspace and Private Link service (ARM template)
2. Tenant 1: Create VNet and VM
3. Tenant 1: Create private endpoint pointing to Tenant 2's Private Link service resource ID (using "Connect by resource ID or alias" method)
4. Tenant 2: Approve the connection in Azure Network Foundation → Pending Connections
5. Tenant 1: Configure private DNS zone (`privatelink.fabric.microsoft.com`) with A record for the workspace FQDN
6. Tenant 2: Deny public access to the workspace (optional — enforces that only the cross-tenant private link can access the workspace)
7. Verify connectivity using `nslookup {workspaceid}.z{xy}.w.api.fabric.microsoft.com` from the VM in Tenant 1

### Security Boundary

> **Important**: Cross-tenant network connectivity does not automatically grant access to workspace resources. Users from Tenant 1 must still:
> - Authenticate with valid credentials in Tenant 2 (via Entra B2B guest account or equivalent)
> - Have the appropriate permissions in Tenant 2's workspace

Network connectivity and identity/authorization are independent. Establishing the private link proves only that network access is possible — not that the user is authorized.

### Tenant-Level Private Link Limitation

Tenant-level private links (from Tenant 1 connecting directly to another tenant's Fabric) are **not supported**. Cross-tenant connectivity only works at the workspace level.

---

## Cross-Workspace Communication (Same Tenant)

See [cross-workspace-communication.md](cross-workspace-communication.md) for the same-tenant scenario where workspace A needs to access workspace B when workspace B has inbound public access restricted.
