# End-to-End Security Scenario

> **Source:** [Microsoft Fabric end-to-end security scenario](https://learn.microsoft.com/en-us/fabric/security/security-scenario)
> Last reviewed: 2026-02-24

This scenario walks through how a healthcare organization in the US would secure a Fabric-based medallion lakehouse architecture (bronze/silver/gold layers). It illustrates how the full set of Fabric security controls work together in practice.

## Architecture Overview

**Goal**: Build a lakehouse using medallion architecture in Fabric for healthcare patient data.

**Layers:**
- **Bronze**: Raw data as-is from source systems
- **Silver**: Data quality checked and transformed; not for general user access
- **Gold**: Aggregated/enriched data for reporting and analysis; access restricted by role and department

**Data sources:**
- On-premises systems (EHR, lab results, wearable devices) — behind firewalls
- Azure SQL Database — behind private endpoints
- Azure Data Lake Storage Gen2 — existing data lakes
- Legacy ETL processes in Azure Data Factory and Azure Databricks

---

## Inbound Protection: Who Can Access Fabric

**Requirement**: All users must use MFA and must be on the corporate network.

**Solution**: Microsoft Entra Conditional Access

- Require MFA for all users
- Restrict access to corporate network IP ranges
- Require Intune-enrolled, compliant devices (OS version, security patches)

**If full network isolation is needed**: Enable tenant-level Private Link and block public internet access. All users then access Fabric only through the VPN or ExpressRoute to the Azure VNet with the private endpoint.

---

## Outbound Protection: Secure Access to Data Sources

### On-premises data (EHR, lab results)

Use **On-Premises Data Gateway** with Data Factory pipelines and/or Dataflow Gen2. Data Factory supports 100+ connectors including healthcare-specific data formats.

> Note: If tenant Private Link is enabled, OPDG will not work. Use VNet data gateway instead.

### Azure SQL Database (behind private endpoints)

Two options depending on workload type:
- **VNet Data Gateway** + Dataflow Gen2 (low-code, no gateway infra to manage)
- **Managed Private Endpoints** + Spark notebooks (code-based ingestion)

### ADLS Gen2 (existing data lake)

Use **Trusted Workspace Access** + **OneLake shortcuts**:
- Create a workspace identity for the Fabric workspace
- Configure resource instance rules on the ADLS Gen2 storage account (via ARM template)
- Create OneLake ADLS shortcuts in the Lakehouse
- All Fabric experiences (Spark, SQL endpoint, pipelines, Power BI, Dataflows) can access the data through the shortcut

### Azure Databricks and Azure Synapse Spark

Use the **OneLake API (ABFS driver)** — Databricks and Synapse Spark can write to OneLake using the same Azure Blob Filesystem interface used with ADLS Gen2.

### Azure Data Factory (existing pipelines)

Redirect output destination using the **Lakehouse connector** in existing ADF pipelines. No need to rebuild the pipelines from scratch.

---

## Access Control: Who Sees What Data

**Three workspaces (one per medallion layer):**

| Workspace | Admin | Contributors | Viewers |
|-----------|-------|-------------|---------|
| Bronze | You (data engineer) | Data engineering team | None |
| Silver | You (data engineer) | Data engineering team | None |
| Gold | You (data engineer) | Data engineering team | Data analysts + business users (restricted) |

**For gold layer:**
- Create **two lakehouses**: one with raw data, one with shortcuts enforcing SQL security (RLS, OLS, column masking)
- Connect a **Direct Lake semantic model** to the first lakehouse
- Configure semantic model with a **fixed identity** and implement **Row-Level Security (RLS)** in the model
- **Share only** the semantic model and the restricted second lakehouse with analysts/business users — not the pipelines, notebooks, or raw lakehouse
- Grant **Build permission** on the semantic model for report creation

---

## Data Handling: At Rest and In Transit

- **Encryption at rest**: All OneLake data encrypted with Microsoft-managed keys by default
- **CMK option**: If CMK is required, either use Workspace Customer Managed Keys (CMK in Azure Key Vault encrypts the Microsoft encryption key per workspace) or keep data in external ADLS Gen2 with CMK enabled and access via OneLake shortcuts

For the shortcut approach with CMK on external storage:
- Data never leaves the CMK-encrypted storage
- Fabric performs in-place reads
- ADLS Gen2 shortcuts support write operations (data written back is also CMK-encrypted)
- Disable shortcut caching for S3, GCS, and S3-compatible shortcuts (cached data persists on OneLake without CMK)

- **In transit**: TLS 1.2 minimum; TLS 1.3 where possible; internal Fabric traffic over Microsoft backbone

---

## Data Residency

- Create a **Multi-Geo capacity** in the required region (e.g., West US for US-based operations)
- Assign workspaces to this capacity
- Compute and storage (including OneLake) reside in the capacity region
- Tenant metadata stays in the home region
- Data only stored/processed in these two geographies

---

## Common Security Scenarios Quick Reference

| Scenario | Tool | Direction |
|---------|------|-----------|
| ETL at scale from on-premises/other cloud, behind firewalls | On-premises data gateway + pipeline copy activity | Outbound |
| Low-code data loading from on-premises via firewall | On-premises data gateway + Dataflow Gen2 | Outbound |
| Azure data behind private endpoints, no gateway infra | VNet data gateway + Dataflow Gen2 | Outbound |
| Spark-based ingestion from Azure behind private endpoints | Fabric notebooks + managed private endpoints | Outbound |
| Redirect existing ADF pipelines to Fabric | Lakehouse connector in ADF | Outbound |
| Redirect Databricks/Synapse Spark to Fabric | OneLake API (ABFS driver) | Outbound |
| Lock down Fabric backend from internet | Already done by Microsoft — add Conditional Access + optional private links | Inbound |
| Restrict Fabric access to corporate network only | Entra Conditional Access | Inbound |
| Require MFA for all Fabric access | Entra Conditional Access | Inbound |
| Lock entire tenant to VNet only | Tenant-level Private Link + Block Public Internet Access | Inbound |
