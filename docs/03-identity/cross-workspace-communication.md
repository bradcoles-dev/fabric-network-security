# Cross-Workspace Communication

> **Source:** [About cross-workspace communication](https://learn.microsoft.com/en-us/fabric/security/security-cross-workspace-communication)
> Last reviewed: 2026-02-24

## The Problem

When workspace B has **restricted inbound public access** (via workspace-level private links), connections from workspace A to workspace B are blocked by default — even if both workspaces are in the same tenant and even if private endpoints exist between the client and workspace A.

The reason: it is the **source workspace** (workspace A) that initiates the connection to workspace B, not the client. The private endpoint between the client and workspace A does not help with this workspace-to-workspace connection.

## Solutions

Two options exist to enable cross-workspace communication when the target workspace restricts inbound access:

### Option 1: Managed Private Endpoint (Recommended for Spark / Data Engineering)

Create a managed private endpoint in the **source workspace** (workspace A) that points to the **Private Link service** of the **target workspace** (workspace B).

This establishes a trust relationship: workspace A is allowed to connect to workspace B through the managed private endpoint.

**Steps:**
1. Get the Azure Private Link service resource ID for the target workspace (find it by viewing the resource JSON for the workspace in Azure)
2. Create a managed private endpoint from workspace A's settings pointing to that resource ID
3. The Private Link service owner for workspace B must approve the request in **Azure Private Link Center → Pending Connections**

**Currently supported cross-workspace scenarios:**
- Shortcut access from one workspace to another workspace
- Notebook in source workspace accessing a lakehouse in target workspace (must use workspace FQDN, not friendly name)
- Pipeline in source workspace accessing a notebook in target workspace
- Eventstream accessing a lakehouse in another workspace
- Eventstream accessing an eventhouse in another workspace (with event processing before ingestion)

### Option 2: Data Gateway

Use a **VNet data gateway** or **on-premises data gateway (OPDG)** to bridge workspace A to workspace B.

Requirements:
- A private endpoint must be created on the VNet holding the data gateway
- This private endpoint points to the Private Link service for the target workspace (workspace B)

The gateway acts as the intermediary that can reach workspace B through the private endpoint.

**Use case**: Primarily for Power BI or Data Factory workloads that need to access data in a restricted workspace.

## Important Behavior Notes

- If the target workspace **allows inbound public access**, connections from other workspaces work without additional configuration
- Managed private endpoints for cross-workspace access are provisioned and managed within the **managed VNet** of the source workspace
- The approval workflow requires the target workspace's Private Link service owner to act in Azure — this is an Azure-side approval, not a Fabric-side action

## Relationship to Outbound Access Protection

If workspace A also has **outbound access protection** enabled, you must additionally configure workspace A's outbound allow list to permit connections to workspace B. The managed private endpoint from workspace A to workspace B serves this purpose (it is both the network path and the outbound allowlist entry).
