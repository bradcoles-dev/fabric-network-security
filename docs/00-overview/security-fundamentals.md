# Microsoft Fabric Security Fundamentals

> **Source:** [Microsoft Learn — Microsoft Fabric security fundamentals](https://learn.microsoft.com/en-us/fabric/security/security-fundamentals)
> Last reviewed: 2026-02-24

## Architecture Overview

Fabric is built on Azure and follows a multi-tenant SaaS architecture. The key components from a security perspective:

1. **User/Client** — browser, Power BI Desktop, SSMS, API client
2. **Microsoft Entra ID** — authenticates every request; issues access tokens
3. **Web Front End** — receives requests, facilitates sign-in, routes to backend; serves front-end content
4. **Metadata Platform** — stores tenant metadata, workspace definitions, authorization info; located in **tenant home region**
5. **Back-End Capacity Platform** — compute operations and customer data storage; located in the **capacity region**

The metadata platform and back-end capacity platform each run in **secured virtual networks managed by Microsoft**. These expose secure endpoints to the internet but are otherwise blocked from public access. Internal communication between services is also restricted by network security rules.

> **Key point for enterprise architects**: The backend is already isolated in Microsoft-managed VNets. You are adding controls on top of this baseline, not building the isolation yourself.

## Authentication

- All Fabric requests authenticate via **Microsoft Entra ID**
- Users receive OAuth 2.0 access tokens upon authentication
- Fabric uses these tokens to perform operations in the user's context
- Service principals and managed identities are also supported

**Signed tokens**: For performance, Fabric sometimes encapsulates authorization info into signed tokens issued by the back-end capacity platform. These include the access token, authorization info, and other metadata.

## Authorization

- All Fabric permissions stored centrally in the **metadata platform**
- Fabric services query this platform on demand to validate requests
- Authorization is always workspace-scoped at the top level, with item-level permissions available below that

## Data Residency

- Each tenant is assigned a **home metadata platform cluster** in a single region
- Tenant metadata (including some customer data) is stored in this home region
- Workspaces can be placed in the same or a different region via **Multi-Geo capacities**
- With Multi-Geo: compute and storage (OneLake + experience storage) are in the capacity region; tenant metadata remains in home region
- Customer data is only stored and processed in these two geographies

> **Note**: For Fabric Trial, Power BI Pro, and PPU workspace types, all customer data is stored in the tenant home region only.

## Data Handling

### Encryption at Rest

- All Fabric data stores encrypted at rest using **Microsoft-managed keys** by default
- Data is never persisted to permanent storage in an unencrypted state
- In-memory processing may be unencrypted, but this is transient
- **Workspace Customer Managed Keys (CMK)**: Encrypt the Microsoft encryption key with your own Azure Key Vault keys at the workspace level

### Encryption in Transit

- All inbound communication enforces **TLS 1.2 minimum**; TLS 1.3 negotiated where possible
- Traffic between Microsoft services routes over the **Microsoft global backbone network**
- Outbound Fabric communication to customer-owned infrastructure requires TLS 1.2+; TLS 1.0 and 1.1 are no longer supported

### OneLake

- Automatically provisioned for every Fabric tenant
- Built on Azure; stores any file type (structured or unstructured)
- All Fabric items (warehouses, lakehouses, etc.) automatically store data in OneLake
- Compatible with ADLS Gen2 APIs and SDKs
- Workspace is the primary security boundary for OneLake data

## Multitenant Isolation

- Fabric platform infrastructure is multitenant with **logical isolation** between tenants
- Platform services never run user-written code
- The application layer ensures tenants can only access data from within their own tenant
- Cross-tenant access requires explicit configuration (see [cross-tenant.md](../03-identity/cross-tenant.md))

## Telemetry

- Fabric platform telemetry is stored in a compliant store
- Designed to comply with data and privacy regulations including EU Data Boundary requirements

## Compliance References

- Microsoft Online Services Terms govern the Fabric service
- Microsoft Trust Center is the primary compliance resource
- Fabric follows Security Development Lifecycle (SDL) practices
- Fabric is compliant with ISO 27001, 27017, 27018, 27701, HIPAA, and more
- See the [Microsoft Azure Compliance Offerings](https://aka.ms/azurecompliance) for the full list of certifications
