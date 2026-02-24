# Conditional Access in Microsoft Fabric

> **Source:** [Microsoft Learn — Conditional access in Fabric](https://learn.microsoft.com/en-us/fabric/security/security-conditional-access)
> Last reviewed: 2026-02-24

## What It Is

Microsoft Entra Conditional Access evaluates a set of policies at authentication time to determine whether a user or service principal is permitted to access Fabric. It is identity-based control, not network-level isolation.

Conditional Access can enforce:

- Multifactor authentication (MFA)
- Device compliance (Intune-enrolled, Intune-managed)
- Network location restrictions (IP ranges, named locations, countries/regions)
- Sign-in risk level restrictions
- Application sensitivity conditions

## How Fabric Uses Conditional Access

When a user connects to Fabric, authentication is handled by Entra ID. Conditional Access policies are evaluated at this point. If a policy is triggered and not satisfied, access is denied before the user reaches Fabric.

Fabric does not support other authentication methods such as account keys or SQL authentication (username/password at the Fabric layer). All access flows through Entra ID, making Conditional Access universally applicable.

## The Multi-Service Dependency — Critical for Configuration

This is the most important operational detail for Conditional Access in Fabric:

**Fabric depends on multiple Azure services internally.** A Conditional Access policy must cover all of them to work correctly. Microsoft recommends a single unified policy covering:

- **Power BI Service**
- **Azure Data Explorer**
- **Azure SQL Database**
- **Azure Storage**
- **Azure Cosmos DB**

If you apply a policy to Power BI Service only, features like Dataflows may break because those features call the other services. If your policy is too restrictive (e.g., blocks all apps except Power BI), some Fabric features will fail silently or with confusing errors.

> **If you already have a Conditional Access policy for Power BI, you must add all five services listed above to that existing policy.** Otherwise, Conditional Access will not operate as intended across Fabric.

## Licensing Requirement

Conditional Access requires **Microsoft Entra ID P1** licenses. These are commonly already available in organizations with Microsoft 365 E3/E5, but should be verified before planning on Conditional Access as the primary control.

## Configuration Steps

1. Sign in to the Azure portal as at least a Conditional Access Administrator
2. Navigate to **Microsoft Entra ID → Security → Conditional Access**
3. Create a new policy
4. Under **Assignments → Users**: select target users or groups
5. Under **Target resources → Cloud apps**: select **all five apps**: Power BI Service, Azure Data Explorer, Azure SQL Database, Azure Storage, Azure Cosmos DB
6. Under **Access controls → Grant**: configure your enforcement (MFA, compliant device, etc.)
7. Set **Enable policy** to On and create

## What Conditional Access Can and Cannot Do

| Can Do | Cannot Do |
|--------|-----------|
| Require MFA for all Fabric access | Block traffic at the network layer |
| Restrict access to specific IP ranges | Target Fabric independently of its dependent Azure services |
| Block access from specific countries/regions | Enforce per-workspace policies |
| Require compliant/Intune-enrolled devices | Prevent authenticated users from accessing Fabric over public internet |
| Enforce sign-in frequency and session controls | Replace Private Links for full network isolation |

## Relationship to Private Links

Conditional Access and Private Links are **complementary**, not alternatives:

- **Conditional Access** controls *who* can access Fabric (identity, device, location)
- **Private Links** control *how* traffic reaches Fabric (network path)

For maximum security, organizations should use both: Conditional Access to enforce identity hygiene and Private Links to restrict network-level access.

Microsoft positions Conditional Access as the right choice when:
- Network-level isolation is not required
- The organization needs flexible, identity-driven access control
- Azure VNet infrastructure is not available or too complex to set up for this purpose

## Trusted Access for On-Premises Scenarios

When data sources are behind firewalls, Fabric does not need to be in the same private network. Features like on-premises data gateways, trusted workspace access, and managed private endpoints enable secure access without putting Fabric compute inside a private network.
