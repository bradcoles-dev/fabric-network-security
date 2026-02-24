# Workspace IP Firewall Rules

> **Sources:**
> - [About workspace IP firewall rules](https://learn.microsoft.com/en-us/fabric/security/security-workspace-level-firewall-overview)
> - [Set up workspace IP firewall rules](https://learn.microsoft.com/en-us/fabric/security/security-workspace-level-firewall-set-up)
> Last reviewed: 2026-02-24

## What It Is

Workspace IP firewall rules let workspace admins define a list of allowed **public IP address ranges** for inbound access to a workspace. Only connections from listed IPs — and connections through workspace-level private endpoints — are permitted. All other public connections are denied.

IP firewall is a lighter-weight alternative to Private Links when full VNet isolation is not feasible or required.

## What It Is Not

- **Not a network-level block**: Traffic still traverses the public internet; IP firewall checks the source IP at the Fabric layer
- **Not outbound control**: IP firewall rules only apply to inbound traffic; outbound connections from the workspace are unaffected
- **Not a replacement for Private Links**: Does not provide the same level of network isolation; traffic is not routed over Microsoft's backbone

## Supported Item Types

IP firewall rules control access to:

- Lakehouse, SQL Endpoint, and Shortcuts
- Direct connections via OneLake endpoint
- Notebooks, Spark Job Definitions, and Environments
- Machine Learning Experiments and Models
- Pipelines, Copy Jobs, Mounted Data Factories
- Warehouses
- Dataflows Gen2 (CI/CD)
- Variable Libraries
- Mirrored Databases (Open Mirroring, Cosmos DB)
- Eventstreams
- Eventhouses

## Limitations and Constraints

| Constraint | Detail |
|-----------|--------|
| Supported capacity types | All Fabric capacity types including Trial (unlike workspace private links) |
| Max rules per workspace | 256 |
| Supported IP types | Public IP addresses only; RFC 1918 private addresses (10.x, 172.16-31.x, 192.168.x) are NOT supported |
| VM IPs on VNets with private endpoints | Cannot be added as IP firewall rules |
| Duplicate names | Not allowed |
| Spaces in IP addresses | Not allowed |
| Outbound traffic | Not affected by IP firewall rules |

## Interaction with Tenant-Level Settings

How you configure and access IP firewall settings depends on the tenant configuration:

| Tenant Private Link | Tenant Public Internet | Workspace Private Link | Portal Access to Firewall Settings? | API Access? |
|--------------------|----------------------|-----------------------|-------------------------------------|------------|
| Enabled | Blocked | Allowed | Yes, from tenant PL network only | Yes, via tenant PL FQDN or api.fabric.microsoft.com |
| Enabled | Blocked | Blocked (public denied) | No | Yes, via tenant PL network |
| Enabled | Allowed | Allowed | Yes, public or tenant PL network | Yes |
| Enabled | Allowed | Blocked | No | Yes |
| Disabled | N/A | Allowed | Yes, public internet | Yes |
| Disabled | N/A | Blocked | No | Yes |

> **Key insight**: When a workspace has its public access blocked via workspace private links, you cannot configure IP firewall settings through the Fabric portal — you must use the REST API.

## Access Behavior with IP Firewall Configured

When tenant Block Public Access is **enabled**: Even an allowed IP in the workspace firewall cannot access the Fabric portal (portal requires tenant private link). But API access (api.fabric.microsoft.com) still works from allowed IPs.

When tenant Block Public Access is **disabled**: An allowed IP can access both the Fabric portal and REST API.

## On-Premises Networks

To allow access from on-premises networks:
1. Identify the internet-facing (NAT) IP addresses your on-premises network uses
2. Contact your network administrator if unsure
3. For ExpressRoute with Microsoft peering, identify the NAT IP addresses provided by your service provider or configured by your team

## Recovery if Locked Out

If incorrect firewall rules make the workspace inaccessible through the portal, use the Fabric API to update IP firewall rules from a permitted network path (e.g., through a tenant-level private link or by temporarily allowing the API endpoint from an admin machine).
