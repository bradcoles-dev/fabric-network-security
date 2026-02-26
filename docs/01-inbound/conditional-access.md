# Conditional Access in Microsoft Fabric

> **Sources:**
> - [Microsoft Learn — Conditional access in Fabric](https://learn.microsoft.com/en-us/fabric/security/security-conditional-access)
> - [What is Conditional Access?](https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview) — Last updated Nov 18, 2025
> - [Plan a Conditional Access deployment](https://learn.microsoft.com/en-us/entra/identity/conditional-access/plan-conditional-access)
> - [Conditional Access service dependencies](https://learn.microsoft.com/en-us/entra/identity/conditional-access/service-dependencies)
> Last reviewed: 2026-02-26

## What It Is

Microsoft Entra Conditional Access is Microsoft's **Zero Trust policy engine**. It evaluates signals from identity, device, location, and risk to make real-time access decisions. It is authentication-layer control — it determines who can access Fabric and under what conditions. It is not network-layer isolation.

Conditional Access policies are enforced **after first-factor authentication completes** (i.e., after the user enters their password) but before full access is granted. If a policy requires MFA, the user is prompted at this point. If a policy requires a compliant device, device state is checked here.

Conditional Access can enforce:

- Multifactor authentication (MFA)
- Device compliance (Intune-enrolled, Intune-managed)
- Named location restrictions (IP ranges, countries/regions)
- Sign-in risk and user risk level (requires P2 — see below)
- Token protection and session controls

## How Fabric Uses Conditional Access

When a user connects to Fabric, authentication is handled by Entra ID. Conditional Access policies are evaluated at this point. If a policy is triggered and not satisfied, access is denied before the user reaches Fabric.

Fabric does not support other authentication methods such as account keys or SQL authentication at the Fabric layer. All access flows through Entra ID, making Conditional Access universally applicable to interactive user access.

## The Multi-Service Dependency — Critical for Configuration

**Fabric depends on multiple Azure services internally.** A Conditional Access policy must cover all of them to work correctly. Microsoft recommends a single unified policy covering:

- **Power BI Service**
- **Azure Data Explorer**
- **Azure SQL Database**
- **Azure Storage**
- **Azure Cosmos DB**

If you apply a policy to Power BI Service only, features like Dataflows may break because those features call the other services. If your policy is too restrictive (e.g., blocks all apps except Power BI), some Fabric features will fail silently or with confusing errors.

> **If you already have a Conditional Access policy for Power BI, you must add all five services listed above to that existing policy.** Otherwise, Conditional Access will not operate as intended across Fabric.

> **Note:** The general Entra Conditional Access service dependencies documentation does not specifically list Fabric or Power BI dependencies. The five-app list above comes from the Fabric-specific Conditional Access documentation. Verify this list is current before implementation — Microsoft may update Fabric's internal service dependencies as the platform evolves.

## Licensing Requirements

| Capability | License Required |
|-----------|-----------------|
| Basic CA (MFA, device compliance, named locations) | Entra ID P1 (included in M365 E3/E5) |
| Risk-based policies (sign-in risk, user risk) | Entra ID P2 (included in M365 E5) |
| Identity Protection signals | Entra ID P2 |
| Token protection | Entra ID P2 |
| Conditional Access for Workload Identities | Entra Workload ID (separate licence — not included in E5) |

Organizations with M365 E5 have P2 and can use the full Conditional Access feature set for user identities, including risk-based policies.

## Named Locations

Named locations are the mechanism for IP-based restrictions. An administrator defines a set of IP address ranges (e.g., corporate VPN egress IPs, ZScaler egress IPs) as a named location, then CA policies can require that users connect from that location.

**Important for ZScaler environments:** If users access Fabric through a ZScaler proxy, the IP address seen by Entra ID is the ZScaler egress IP, not the user's corporate IP. Named location rules must be based on ZScaler's egress IP ranges, not internal corporate ranges.

## Risk-Based Policies (P2 / E5)

With Entra ID P2, Conditional Access can evaluate real-time risk signals from Microsoft Entra ID Protection:

- **Sign-in risk**: Unusual sign-in patterns, anonymous IP, atypical travel
- **User risk**: Leaked credentials, suspicious activity patterns

Policies can require step-up authentication or block access entirely based on risk level (low, medium, high). This is available to clients with M365 E5 at no additional cost.

## Service Principals and Workload Identities

Standard Conditional Access policies (targeting users and groups) **do not automatically cover service principals**. Service-to-service connections — such as OPDG service accounts, Fabric pipeline service principals, or external application integrations — are not governed by user CA policies.

Protecting service principals requires **Conditional Access for Workload Identities**, which is a separate feature requiring the **Entra Workload ID** licence (not included in E5). Without this, service principals can authenticate to Fabric-dependent services regardless of user CA policies.

> **Operational implication:** If a CA policy is intended to restrict all access to Fabric, service principals used by OPDG, pipelines, or other automated processes may bypass it. Audit which service principals have access to the five Fabric-dependent apps before enforcing restrictive CA policies.

## Configuration Steps

1. Sign in to the Microsoft Entra admin center as at least a Conditional Access Administrator
2. Navigate to **Entra ID → Security → Conditional Access**
3. Create a new policy — enable in **report-only mode first** (see deployment note below)
4. Under **Assignments → Users**: select target users or groups; **exclude emergency access accounts** (see below)
5. Under **Target resources → Cloud apps**: select **all five apps**: Power BI Service, Azure Data Explorer, Azure SQL Database, Azure Storage, Azure Cosmos DB
6. Under **Conditions**: configure named locations, device platforms, or risk levels as required
7. Under **Access controls → Grant**: configure enforcement (MFA, compliant device, etc.)
8. Monitor sign-in logs in report-only mode for at least one week before switching to enforcement

## Operational Requirements

### Emergency / Break-Glass Accounts
**All CA policies must exclude emergency access accounts.** These are accounts held outside normal identity governance processes, used only if normal admin access is lost (e.g., MFA outage, identity provider failure). If CA policies apply to break-glass accounts and authentication methods fail, the entire tenant can be locked out. Emergency accounts must be excluded from every policy without exception.

### Report-Only Mode
Before enforcing any CA policy, enable it in **report-only mode** for a minimum of one week. Review sign-in logs to identify users, service principals, or scenarios that would be blocked. Unexpected failures in report-only mode are informational; the same failures in enforcement mode lock users out. The What-If tool can supplement report-only testing for specific scenarios.

### Policy Limit
There is a **maximum of 195 CA policies per tenant** (across all states: enabled, disabled, report-only). For large Fabric deployments with many applications and user groups, design policies to cover multiple apps in a single policy rather than creating per-app policies.

### Rollback
Disabled policies can be re-enabled immediately. Deleted policies enter a 30-day soft-delete period and can be recovered within that window.

## What Conditional Access Can and Cannot Do

| Can Do | Cannot Do |
|--------|-----------|
| Require MFA for all Fabric access | Block traffic at the network layer |
| Restrict access to specific IP ranges (named locations) | Target Fabric independently of its dependent Azure services |
| Block access from specific countries/regions | Enforce per-workspace policies |
| Require compliant/Intune-enrolled devices | Prevent authenticated users from accessing Fabric over the public internet |
| Enforce sign-in risk and user risk controls (P2) | Replace Private Links for full network isolation |
| Enforce token protection and session controls (P2) | Govern service principal access without Entra Workload ID licence |
| Enforce sign-in frequency and session controls | — |

## Limitations Specific to Fabric

### No per-workspace granularity
CA policies apply to the application (Power BI Service, etc.), not to individual workspaces. You cannot enforce different conditions for different workspaces — for example, requiring a compliant device for PROD Gold but allowing DEV with MFA only. The policy applies uniformly across all workspaces for the targeted apps. Workspace-level Private Link can isolate individual workspaces; CA cannot.

### Service principals are not covered
Standard user CA policies do not apply to service principals. Automated workloads — OPDG service accounts, pipeline service principals, application registrations — authenticate outside the policy and are not subject to its conditions. Covering service principals requires **Conditional Access for Workload Identities**, which requires the **Entra Workload ID licence** (not included in E5). Without it, service principals can authenticate to Fabric-dependent services regardless of user CA policy configuration.

### The five-app dependency list may be incomplete or change
The five Fabric-dependent applications (Power BI Service, Azure Data Explorer, Azure SQL Database, Azure Storage, Azure Cosmos DB) are sourced from Fabric-specific CA documentation, not from the general Entra service dependency catalogue. If Microsoft adds a new internal dependency and Fabric begins calling a sixth service, the CA policy would have a gap with no automatic notification. This list should be re-verified periodically and whenever significant Fabric features are adopted.

### Named location maintenance
If ZScaler egress IPs are used as a named location, ZScaler periodically changes its egress IP ranges for routing purposes. If the named location is not updated when this happens, users are blocked without obvious cause. Named location IP ranges must be treated as a live configuration item, not set-and-forget.

### CA gates access but does not govern authenticated sessions
Once a user satisfies the CA policy and receives an access token, CA has no further visibility into what they do. Large data exports, bulk downloads, and cross-workspace access are all permitted for an authenticated user. Microsoft Defender for Cloud Apps (MDCA) session controls can add behavioural monitoring and controls on top of CA, but this is a separate product requiring additional configuration.

### Token validity window
CA is evaluated at sign-in. If a user's device becomes non-compliant or their risk level changes during an active session, the existing access token remains valid until it expires — up to the configured token lifetime. Continuous Access Evaluation (CAE) can shorten this window by enabling near-real-time revocation, but whether Fabric fully supports CAE is not documented.

---

## Relationship to Private Links

Conditional Access and Private Links are **complementary**, not alternatives:

- **Conditional Access** controls *who* can access Fabric (identity, device, location, risk)
- **Private Links** control *how* traffic reaches Fabric (network path)

For maximum security, organisations should use both. However, CA alone is a valid and strong security posture when a hard network-layer isolation requirement (Private Link) has not been established — particularly when combined with compliant device requirements, named location restrictions, and risk-based policies.

Microsoft positions Conditional Access as the right choice when:
- Network-level isolation is not required
- The organisation needs flexible, identity-driven access control
- Azure VNet infrastructure is not available or not justified

## Trusted Access for On-Premises Scenarios

When data sources are behind firewalls, Fabric does not need to be in the same private network. Features like on-premises data gateways, trusted workspace access, and managed private endpoints enable secure access without putting Fabric compute inside a private network.
