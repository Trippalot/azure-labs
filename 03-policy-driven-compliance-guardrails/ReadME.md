# Policy-Driven Compliance Guardrails

> Enforcing resource tagging and protecting critical resources with Azure Policy and resource locks.

## Overview

Lab 02 established who can do what in the environment via RBAC. This lab establishes *what gets deployed and how it stays deployed* via Azure Policy and resource locks. The two layers work together: RBAC controls the identity surface, Policy controls the configuration surface.

The work covers four areas: applying tags as governance metadata, using Azure Policy to enforce that tags exist on new resources, using Azure Policy to remediate existing resources that are missing tags, and using resource locks to prevent destructive operations on resources that shouldn't be modified or deleted casually.

## Business Scenario

The organization's Azure footprint has grown to the point where untagged resources are causing real operational pain. A recent audit surfaced that a significant portion of resources have no owner attribution, no project association, and no cost center mapping. This breaks three things at once:

- **Cost allocation.** Finance can't split cloud spend by business unit without cost center tags.
- **Incident response.** When a resource misbehaves, the on-call engineer can't quickly identify who owns it.
- **Lifecycle management.** Without project tags, there's no signal for when a resource is safe to decommission.

The remediation plan is to enforce tagging via Azure Policy going forward, backfill existing resources via a policy with a `Modify` effect, and add resource locks to protect resources that should not be deleted accidentally.

## Architecture

```
mg-corp                                  (Management group from Lab 02a)
│
└── Subscription
    │
    └── rg-governance-prod-eastus        (Resource group)
        │
        ├── Tags
        │   └── Cost Center = IT-1001
        │
        ├── Policy assignments
        │   ├── Require Cost Center tag and value (Deny effect)
        │   └── Inherit Cost Center tag from RG (Modify effect, managed identity)
        │
        └── Locks
            └── lock-delete-governance-rg (CanNotDelete)
```

## What I Built

### Task 1 — Resource group with governance tag

Created a resource group and applied a baseline cost allocation tag at provisioning time:

| Setting | Value |
|---|---|
| Resource group name | `rg-governance-prod-eastus` |
| Location | East US |
| Tag name | `Cost Center` |
| Tag value | `IT-1001` |

Tagging at provisioning time matters because policies that rely on tag presence (cost allocation, automated lifecycle management, access controls keyed to tags) need the tag to exist before downstream automation runs against the resource.

### Task 2 — Enforce tagging with the "Require a tag and its value on resources" policy

Assigned the built-in `Require a tag and its value on resources` policy to the resource group:

| Setting | Value |
|---|---|
| Policy definition | Require a tag and its value on resources (built-in) |
| Scope | `rg-governance-prod-eastus` |
| Assignment name | `Require Cost Center tag and its value on resources` |
| Tag name parameter | `Cost Center` |
| Tag value parameter | `IT-1001` |
| Policy enforcement | Enabled |

Tested the policy by attempting to create a storage account without the required tag. The deployment failed at validation with `RequestDisallowedByPolicy`, confirming the `Deny` effect was active. This is the right effect for hard governance requirements where non-compliant resources should never be created in the first place.

### Task 3 — Remediate existing resources with the "Inherit a tag from the resource group" policy

After deleting the first assignment, assigned the built-in `Inherit a tag from the resource group if missing` policy to the same resource group:

| Setting | Value |
|---|---|
| Policy definition | Inherit a tag from the resource group if missing (built-in) |
| Scope | `rg-governance-prod-eastus` |
| Tag name parameter | `Cost Center` |
| Effect | Modify |
| Managed identity | System-assigned, created at assignment time |
| Remediation task | Enabled |

Tested by creating a storage account in the resource group without specifying the Cost Center tag. The resource was created and the tag appeared on it automatically via the remediation task. This validates the `Modify` effect end-to-end and confirms the managed identity has the permissions it needs to write tag values on child resources.

### Task 4 — Resource lock on the governance RG

Applied a `CanNotDelete` lock to the resource group:

| Setting | Value |
|---|---|
| Lock name | `lock-delete-governance-rg` |
| Lock type | CanNotDelete |
| Scope | `rg-governance-prod-eastus` |

Tested by attempting to delete the resource group. The operation was blocked with a lock conflict error, confirming the lock takes precedence over delete permissions held at the subscription or management group scope.

## Security Considerations

**Azure Policy is a pre-deployment control. RBAC and locks are post-deployment.** These are complementary, not redundant. RBAC decides whether an identity is *allowed* to attempt an action. Policy decides whether the resulting resource configuration is *acceptable*. Locks decide whether an existing resource can be modified or destroyed. A complete governance posture uses all three: an authorized identity (RBAC) creates a resource that meets standards (Policy) and is protected from accidental destruction (Lock).

**Scope choice has compounding effects.** This lab assigns policies at the resource group scope to keep the blast radius small for testing. In a production tenant, the same policies belong at the management group scope (specifically at `mg-corp`) so they inherit down to every subscription and resource group automatically. The tradeoff is that a misconfigured policy at management group scope can break deployments tenant-wide, which is why production policy deployments should start in `Audit` mode (which logs non-compliance without blocking) before being promoted to enforcement.

**The Modify effect requires a managed identity, and that identity is privileged.** When Azure Policy modifies resources to enforce compliance, it needs an identity to act under. The system-assigned managed identity created during assignment receives the `Contributor` role on the assignment scope by default, which is broad. For tighter security postures, the role granted to the policy's identity should be the most restrictive built-in role that includes the specific permissions the policy needs (often `Tag Contributor` is enough for tag-related policies). The principle is the same as for any service account: the automation should not run with more authority than the work requires.

**Resource locks override permissions, including Owner-level permissions.** This is what makes them effective for protecting critical infrastructure, and also what makes them a foot-gun if they're left in place after they're no longer needed. A common operational failure pattern is: lock applied during a sensitive migration, migration completes, lock forgotten, six months later someone needs to delete the resource and can't figure out why. Locks should be treated like firewall rules: documented, reviewed, and removed when their justification expires.

**Tags are a security control, not just a governance control.** Cost Center, Owner, and Environment tags inform incident response when a resource generates a security alert. The first question during incident triage is usually "who owns this thing" and the answer needs to come back in seconds, not days. Policy-enforced tagging is what makes that possible at scale.

**Initiatives are the production pattern, not individual policy assignments.** This lab assigns single policies one at a time, which is fine for learning the mechanics. A production governance program groups related policies into an *Initiative* (a policy set) so they can be deployed and versioned as a unit. Built-in initiatives like `Azure Security Benchmark` and `NIST SP 800-53 R5` bundle dozens of policies into a single assignment, which is the right pattern for compliance-driven environments.

## References

- [Azure Policy overview](https://learn.microsoft.com/azure/governance/policy/overview)
- [Built-in policy definitions](https://learn.microsoft.com/azure/governance/policy/samples/built-in-policies)
- [Policy effects reference](https://learn.microsoft.com/azure/governance/policy/concepts/effects)
- [Resource locks overview](https://learn.microsoft.com/azure/azure-resource-manager/management/lock-resources)
- [Tagging best practices](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-tagging)
