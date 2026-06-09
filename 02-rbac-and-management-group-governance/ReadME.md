# RBAC and Management Group Governance

> Establishing a management group hierarchy and applying least-privilege role assignments for a support team.

## Overview

This lab extends the identity foundation from Lab 01 into the authorization layer. With a security group already provisioned, the next step is to scope what that group can actually do — and where. This lab creates a management group to centralize governance across subscriptions, assigns a built-in Azure role to the group, and builds a custom role that removes a specific high-impact permission from a cloned baseline to enforce least privilege.

Management groups are the highest level in the Azure resource hierarchy and are where governance decisions (RBAC, Azure Policy, cost controls) should live whenever they need to apply across more than one subscription. Putting RBAC and Policy at the management group level means inheritance handles enforcement automatically as subscriptions are added or moved.

## Business Scenario

The organization is consolidating Azure subscriptions under a unified governance structure. A support team needs scoped permissions across all subscriptions to perform two functions:

- Create and manage virtual machines (operational support)
- Open Azure support tickets with Microsoft

The team must **not** be able to register new Azure resource providers, which is a tenant-level action that can expose new services and unmanaged attack surface. The built-in `Support Request Contributor` role includes the resource provider registration permission, so it can't be used as-is — a custom role is required to remove that specific action while preserving the rest of the role's scope.

The architecture target: one management group containing all subscriptions, with the support team's security group (`sg-eng-lab-admins` from Lab 01) holding scoped roles inherited down to every subscription in the hierarchy.

## Architecture

```
Tenant Root Management Group
│
└── mg-corp                              (Management group)
    │
    ├── Role assignments
    │   ├── sg-eng-lab-admins → Virtual Machine Contributor
    │   └── sg-eng-lab-admins → Custom Support Request (no provider registration)
    │
    └── Subscription(s)                  (Inherit role assignments from mg-corp)
```

## What I Built

### Task 1 — Management group hierarchy

Created a management group beneath the tenant root to serve as the governance anchor for all subscriptions:

| Setting | Value |
|---|---|
| Management group ID | `mg-corp` |
| Display name | `mg-corp` |
| Parent | Tenant Root Group |

In a real environment, this single management group would be the top of a deeper hierarchy (e.g., `mg-corp` → `mg-prod` / `mg-nonprod` → subscriptions). For this lab, a single layer is sufficient to demonstrate inheritance.

### Task 2 — Built-in role assignment

Assigned the built-in `Virtual Machine Contributor` role to the `sg-eng-lab-admins` security group at the `mg-corp` scope:

| Setting | Value |
|---|---|
| Scope | `mg-corp` (management group) |
| Role | Virtual Machine Contributor (built-in) |
| Assigned to | `sg-eng-lab-admins` (Entra ID security group) |

The `Virtual Machine Contributor` role permits VM management without granting access to the VM's operating system, networking, or storage — a clean fit for support team responsibilities. Assignment is made to the group, not to individual users, so role membership tracks group membership automatically.

### Task 3 — Custom role derived from Support Request Contributor

Created a custom RBAC role by cloning `Support Request Contributor` and excluding the resource provider registration permission:

| Setting | Value |
|---|---|
| Custom role name | `Custom Support Request` |
| Description | Support ticket creation without resource provider registration. |
| Baseline | Cloned from `Support Request Contributor` |
| Excluded permission | `Microsoft.Support/register/action` (added as `NotAction`) |
| Assignable scope | `mg-corp` |

The role JSON reflects the customization with the excluded action listed under `NotActions`, while all other inherited actions remain in `Actions`. The role is assignable only within `mg-corp` and its descendants, scoping the blast radius if the role is ever assigned in error.

### Task 4 — Activity log review

Verified role assignment events in the `mg-corp` Activity Log, filtering for role-related operations (`Microsoft.Authorization/roleAssignments/write` and custom role definition events). The Activity Log is the authoritative audit trail for RBAC changes at the management group scope.

## Security Considerations

**Least privilege through subtraction, not addition.** Building a custom role from scratch is error-prone — it's easy to forget a permission and break the user experience. Cloning a built-in role and removing the one permission that violates least privilege is the safer pattern. The `NotActions` field is the right tool for this: it preserves the original role's intent while explicitly denying the specific action that creates risk. In this case, `Microsoft.Support/register/action` is the exact action that lets a principal enable new resource providers tenant-wide, which can expose services that haven't been reviewed by the security team.

**Why management group scope, not subscription scope.** Assigning roles at the subscription level means re-doing the assignment every time a new subscription is added. Assigning at the management group level means inheritance handles it. The tradeoff is blast radius — a misconfigured role at `mg-corp` affects every subscription below it — which is why custom roles assignable at this scope need extra review before deployment.

**Group-based assignments are the audit-friendly pattern.** The Activity Log records the role assignment as targeting a group object ID. When group membership changes, the role assignment doesn't change — meaning the audit trail stays clean and the change record lives in Entra ID group membership history rather than in RBAC events. This separation matters for access reviews and for incident response.

**Resource provider registration is a privilege escalation path.** It looks like an innocuous administrative action, but registering a new resource provider can enable services that weren't part of the original threat model (Container Registry, IoT Hub, Cognitive Services, etc.) and create new attack surface that bypasses existing controls. Removing this permission from the support team's role is a deliberate hardening choice, not a cosmetic one.

**The root management group is not a deployment target.** The tenant root group exists, and it's tempting to assign policies and roles there for maximum coverage. Doing so is rarely the right move — assignments at root scope affect every subscription in the tenant including ones that may be exempt for compliance, billing, or sovereignty reasons. `mg-corp` as a top-level child of root preserves the option to add separate management groups (e.g., sandbox, partner, regulated workloads) outside its inheritance path.

**Activity Log retention is short by default.** The management group Activity Log retains events for 90 days. For audit and forensics requirements that exceed this, events should be exported to a Log Analytics workspace via a Diagnostic Setting — typically the same workspace used for Microsoft Sentinel ingestion. This is the foundation for detection engineering around RBAC abuse.

## References

- [Azure built-in roles](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles)
- [Azure custom roles](https://learn.microsoft.com/azure/role-based-access-control/custom-roles)
- [Management groups overview](https://learn.microsoft.com/azure/governance/management-groups/overview)
- [Azure RBAC best practices](https://learn.microsoft.com/azure/role-based-access-control/best-practices)
