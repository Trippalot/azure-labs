# Identity Lifecycle Management in Microsoft Entra ID

> Provisioning users and groups for a new engineering team under a least-privilege identity model.

## Overview

This lab implements the foundational identity layer for a new engineering team being onboarded into a corporate Azure tenant. The objective is to provision user accounts (internal and external), create a security group for the team, and lay the groundwork for downstream access control decisions (RBAC, Conditional Access, and group-based licensing) — all while applying identity governance principles consistent with a Zero Trust posture.

Microsoft Entra ID is the control plane for every authentication and authorization decision in Azure. Getting identity right at provisioning time prevents most of the access control problems that surface later in an environment.

## Business Scenario

A mid-sized organization is standing up a pre-production lab environment for an engineering team. A small group of engineers (including one external contractor) will be responsible for managing the lab's virtual machines and supporting services. Identity requirements:

- Internal engineers must authenticate using corporate Entra ID accounts
- The external contractor must access the environment via B2B guest invitation rather than a full corporate identity
- All engineers must be grouped together so that downstream RBAC assignments can be applied to the group rather than to individual users
- The group should be structured to support future automation (dynamic membership, group-based access reviews)

## Architecture

```
Microsoft Entra ID Tenant
│
├── Users
│   ├── jsmith.contoso          (internal, IT Lab Administrator)
│   └── contractor@external.com (B2B guest, IT Lab Administrator)
│
└── Groups
    └── sg-eng-lab-admins       (Security group, Assigned membership)
        ├── Owner: <admin>
        └── Members: jsmith.contoso, contractor@external.com
```

## What I Built

### Task 1 — Internal user provisioning

Created an internal Entra ID user account with role-relevant attributes populated at creation time:

| Setting | Value |
|---|---|
| User principal name | `jsmith.contoso` |
| Display name | `Jordan Smith` |
| Job title | `IT Lab Administrator` |
| Department | `IT` |
| Usage location | United States |
| Password | Auto-generated |
| Account enabled | True |

Populating `Job title` and `Department` at provisioning time is deliberate — these attributes become the matching criteria for dynamic group membership (Premium P1/P2) and are commonly referenced by Conditional Access policies and access reviews.

### Task 2 — External user invitation (B2B)

Invited an external collaborator via the Entra ID B2B invitation flow. The guest is treated as a first-class identity in the tenant and inherits the same attribute schema as internal users, but the source of truth for authentication remains the guest's home tenant.

### Task 3 — Security group with assigned membership

Created a security group with both the internal user and the external guest as members:

| Setting | Value |
|---|---|
| Group name | `sg-eng-lab-admins` |
| Group type | Security |
| Membership type | Assigned |
| Owner | Self (admin) |
| Members | Internal user + B2B guest |

The `sg-` prefix follows the common enterprise naming convention distinguishing security groups (used for permissioning) from Microsoft 365 groups (used for collaboration). Membership is set to Assigned for this lab; the design intent is to migrate to Dynamic membership (`user.jobTitle -eq "IT Lab Administrator"`) once Entra ID P1 licensing is available.

## Security Considerations

This is a basic provisioning lab on its surface, but identity decisions made at this layer cascade into every access control decision downstream. Considerations that informed the design:

**Group-based access, not user-based.** Assigning Azure RBAC roles directly to users is technically possible but operationally fragile — it doesn't survive turnover, it bypasses access reviews, and it scales poorly. Building the security group first means every downstream RBAC assignment in this portfolio targets the group, not the individual.

**Guest user risk.** B2B guests are convenient but expand the trust boundary of the tenant. The guest's authentication is governed by their *home* tenant's policies (MFA, password complexity, conditional access), not this tenant's. Production hardening would include: restricting guest invitation rights, enabling Entra ID External Identities access reviews, applying Conditional Access policies that specifically target guest users, and limiting which directory objects guests can enumerate via the External collaboration settings.

**Attribute-driven authorization.** Populating `Job title` and `Department` at provisioning isn't cosmetic — these are the inputs to dynamic group rules, access package policies, and Conditional Access user filters. Identity governance only works if the underlying attribute data is clean.

**Lifecycle and joiner-mover-leaver.** This lab handles the "joiner" phase. A complete identity lifecycle program also requires defined processes for "mover" (role changes triggering group membership recalculation) and "leaver" (deprovisioning, license reclamation, guest user expiration). Entra ID Governance access reviews and Lifecycle Workflows address this in P2 tenants.

**Least privilege at the identity layer.** No roles or permissions are granted in this lab. That's intentional — identity creation should be decoupled from authorization. The group is the unit of authorization, and it gets its permissions in the next lab.

## References

- [Microsoft Entra ID documentation](https://learn.microsoft.com/entra/identity/)
- [B2B collaboration overview](https://learn.microsoft.com/entra/external-id/what-is-b2b)
- [Dynamic group membership rules](https://learn.microsoft.com/entra/identity/users/groups-dynamic-membership)
