# Azure Identity & Governance Lab

## Overview

I built this lab to practice the identity and governance work that comes with managing Azure resources.

Instead of assigning permissions directly to individual users, I used Microsoft Entra groups and Azure RBAC. I also worked with a custom role, Azure Policy, resource locks, and Bicep.

The main goal was to answer a few practical questions:

- How do I give developers the access they need without giving them too much?
- How can I give a compliance user visibility without letting them make changes?
- How does RBAC inheritance work?
- How can Azure Policy stop resources from being created incorrectly?
- How do resource locks protect important resources?
- Can I rebuild the same setup with Infrastructure as Code?

---

## Architecture

```text
Microsoft Entra ID
        |
        +-- grp-lab-developers
        |      |
        |      +-- Contributor
        |
        +-- grp-lab-compliance
               |
               +-- Reader
               +-- Lab Tag Operator

Azure Subscription
        |
        +-- rg-iam-lab
        |      |
        |      +-- Azure RBAC
        |      +-- Azure Policy
        |      +-- CanNotDelete lock
        |
        +-- rg-iam-lab-iac
               |
               +-- Bicep rebuild
```

---

## 1. Group-Based RBAC

I used Entra groups instead of assigning roles directly to individual users.

This makes access easier to manage because permissions stay attached to the group. Users can be added or removed from the group without changing the Azure role assignments themselves.

![Group members](screenshots/group%20members.png)

The developer group received **Contributor** access at the resource-group level.

![Developer permissions](screenshots/developer%20permissions.png)

I tested the developer account by deploying a resource into the resource group.

![Developer deployment success](screenshots/developer-deployment-success.png)

The compliance group received **Reader**, which allowed the auditor to view resources without changing them.

![Auditor effective access](screenshots/auditor%20effective%20access.png)

I also tested a write operation with the auditor account and confirmed that it was denied.

![Auditor write denied](screenshots/12-auditor-write-denied.png)

---

## 2. Custom Role

I created a custom role called **Lab Tag Operator**.

The role allows the user to:

- read Azure resources
- write resource tags

It does not give the user broad Contributor permissions.

The custom role included:

```text
*/read
Microsoft.Resources/tags/write
```

![Custom role JSON](screenshots/custom-role-json.png)

I assigned the role to the compliance group.

![Tag Operator assigned](screenshots/tag-operator-assigned.png)

This gave the auditor enough access to work with tags without giving the account permission to manage the rest of the resource.

---

## 3. Azure Policy

I assigned a policy that requires resources to include an `Environment` tag.

I then intentionally tried to deploy a resource without the required tag.

Azure blocked the deployment.

![Policy assignment](screenshots/policy-assignment.png)

![Policy denied deployment](screenshots/policy-denied.png)

After adding the required tag, the deployment succeeded.

![Tagged resource success](screenshots/tagged-success.png)

This was a useful example of the difference between **RBAC** and **Azure Policy**.

RBAC answers:

> Who is allowed to do something?

Policy answers:

> What configurations are allowed?

A user can have permission to create a resource and still be blocked by Policy if the resource does not meet the organization's rules.

---

## 4. Resource Locks

I added a **CanNotDelete** lock to protect the resource group from accidental deletion.

![Resource lock](screenshots/delete%20lock.png)

I then tested deleting the protected resource and confirmed that Azure blocked the operation.

![Lock blocks delete](screenshots/lock-blocks-delete.png)

The Activity Log also recorded the failed operation.

![Activity log](screenshots/activity-log-scopelocked.png)

This helped reinforce that resource locks are separate from RBAC.

Having a role such as Contributor does not automatically mean a user can bypass a resource lock.

---

## 5. Rebuilding the Lab with Bicep

After configuring the environment through the Azure portal, I rebuilt the core setup using Bicep.

The Bicep deployment created a separate resource group:

`rg-iam-lab-iac`

The Infrastructure as Code version included the main governance controls from the lab:

- resource group
- group-based role assignments
- custom role
- tag policy
- resource lock

I validated the deployed configuration after the deployment completed.

![IaC configuration verified](screenshots/iac-config-verified.png)

The Bicep version gave me practice moving from portal-based administration to a repeatable deployment.

---

## Validation Summary

| What I configured | How I tested it |
|---|---|
| Developer group RBAC | Developer successfully deployed a resource |
| Compliance Reader access | Auditor could view resources |
| Least privilege | Auditor write operation was denied |
| Custom Tag Operator role | Compliance group received tag-specific permissions |
| Required-tag Policy | Untagged resource deployment was blocked |
| Policy compliance | Tagged deployment succeeded |
| Resource lock | Delete operation was blocked |
| Infrastructure as Code | Governance configuration rebuilt with Bicep |

---

## Troubleshooting

### RBAC Inheritance

One area I spent time working through was RBAC scope and inheritance.

A role assigned at the resource-group level applies to the resources below that resource group.

That means I did not need to assign the same role individually to every resource.

### Reader vs. Contributor

I also had to correct one of my original role assignments.

The developer group initially had both Reader and Contributor.

That was unnecessary because Contributor already includes the ability to read resources.

I removed the redundant Reader assignment and kept Contributor for the developer group.

### Policy vs. Permissions

Another useful lesson was that having permission to deploy something does not mean Azure has to accept the configuration.

The account could have enough RBAC access to create a resource, but Policy could still deny the deployment if the required tag was missing.

---

## What I Learned

The biggest things I took away from this lab were:

- Group-based RBAC is easier to manage than assigning roles user by user.
- Azure RBAC permissions are inherited down the resource hierarchy.
- Contributor and Reader can overlap, so assigning both is usually unnecessary.
- Custom roles are useful when built-in roles give more access than needed.
- Azure Policy controls configuration, while RBAC controls permissions.
- Resource locks provide another layer of protection against accidental changes or deletion.
- Bicep makes the same configuration repeatable instead of relying only on portal clicks.

---

## Scope

This lab focused on core Azure identity and governance features.

It did not include:

- Privileged Identity Management
- access reviews
- management groups
- Conditional Access
- enterprise-scale policy initiatives
- production naming or tagging standards

Those would make sense in a larger identity/governance project.

---

## Cleanup

After testing, I removed the lab resources that were no longer needed.

Microsoft Entra users and groups were separate from the resource-group cleanup and were managed independently.

---

## Repository Structure

```text
azure-identity-governance/
├── README.md
├── bicep/
│   └── main.bicep
└── screenshots/
```

---

## Skills Used

- Microsoft Entra ID
- Azure RBAC
- Azure role assignments
- RBAC scopes and inheritance
- Custom roles
- Azure Policy
- Resource locks
- Azure Activity Log
- Bicep
- Azure Portal
- Azure CLI
