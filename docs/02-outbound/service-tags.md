# Azure Service Tags for Microsoft Fabric

> **Source:** [Service tags — Microsoft Fabric](https://learn.microsoft.com/en-us/fabric/security/security-service-tags)
> Last reviewed: 2026-02-24

## What They Are

Azure service tags are named groups of IP address ranges for Azure services that are automatically maintained by Microsoft. They simplify NSG and firewall rule management — instead of tracking Fabric IP ranges manually, you reference a service tag.

Service tags are used to **allow or block traffic to/from Fabric** at the Azure networking layer (NSGs, Azure Firewall, user-defined routes).

## Supported Service Tags for Fabric

| Tag | Purpose | Direction | Regional? | Azure Firewall? |
|-----|---------|-----------|-----------|----------------|
| `DataFactory` | Azure Data Factory | Both (inbound + outbound) | Yes | Yes |
| `DataFactoryManagement` | On-premises pipeline activity | Outbound only | No | Yes |
| `EventHub` | Azure Event Hubs | Outbound only | Yes | Yes |
| `Power BI` | Power BI and Microsoft Fabric | Both | Yes | Yes |
| `PowerQueryOnline` | Power Query Online | Both | No | Yes |
| `KustoAnalytics` | Real-Time Analytics (RTI/KQL) | Both | No | **No** |
| `SQL` | Warehouse (SQL endpoint) | Outbound only | Yes | Yes |

> **Important**: There is **no service tag for untrusted code** used in Data Engineering (Spark) items. Spark workloads do not have a service tag.

## How to Use Service Tags

Service tags can be applied in:
- **Network Security Groups (NSGs)**: Define inbound/outbound rules using service tag names instead of IP ranges
- **Azure Firewall**: Use service tags in network rules or application rules
- **User-defined routes (UDRs)**: Direct traffic appropriately

### Common Use Case: Allow Azure SQL MI Inbound from Fabric

To allow an Azure SQL Managed Instance to accept connections from Fabric:
1. Go to the NSG associated with the SQL MI subnet
2. Add an inbound rule allowing traffic from the `Power BI` service tag on port 1433
3. Optionally scope to the regional variant (e.g., `Power BI.WestUS`)

### Common Use Case: Allow Fabric Users to Access SQL Connection Strings

To allow users on a corporate VM to connect to Fabric SQL connection strings (e.g., `<guid>.datawarehouse.fabric.microsoft.com`) while blocking other public internet traffic:
1. Add an outbound rule allowing traffic to the `Power BI` service tag
2. Block other outbound internet traffic

## Getting the IP Address Ranges

For systems that don't support service tags natively (e.g., on-premises firewalls), you can get the IP ranges as a JSON file or programmatically:
- Download: [Azure IP Ranges and Service Tags JSON](https://www.microsoft.com/download/details.aspx?id=56519)
- PowerShell: `Get-AzNetworkServiceTag -Location <region>`
- Azure CLI: `az network list-service-tags --location <region>`
- REST API: Service Tags Discovery API

> IP ranges change periodically. Do not hardcode them — use service tags where possible, or automate JSON downloads for on-premises firewalls.

## Limitations

- `KustoAnalytics` (RTI) cannot be used with Azure Firewall application rules
- No service tag exists for Spark/Data Engineering untrusted code compute
- Service tags represent Fabric's outbound IP ranges, not guarantees about specific request sources — any service using the same Azure infrastructure may share the same service tag ranges
