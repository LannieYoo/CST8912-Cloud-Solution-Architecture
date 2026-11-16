# Graded Lab Activity #8

**Governance for Azure resources: tags, Azure Policy and resource locks**

| | |
|---|---|
| **Name** | Hye Ran Yoo |
| **Student number** | 041145212 |
| **Course** | CST8912 – Cloud Solution Architecture |
| **Section** | 013 |
| **Date** | November 16, 2026 |

## What this lab does

I used a tag, two Azure policies and a resource lock on one resource group. The lab is about the governance principle for tagging Azure resources.

1. Assign tags via the Azure portal (Task 1). I create the resource group `CST8912` with the tag `Cost Center : 000`.
2. Enforce tagging via an Azure policy (Task 2). The policy requires the tag. I try to create a storage account without the tag, and Azure stops it.
3. Apply tagging via an Azure policy (Task 3). I replace the policy. The new policy copies the tag from the resource group. I create a storage account without the tag, and the tag is added for me.
4. Configure and test resource locks (Task 4). I add a Delete lock to the resource group and try to delete the group.
5. Clean up the resources (Task 5).

```mermaid
%%{init: {"flowchart": {"nodeSpacing": 30, "rankSpacing": 34, "padding": 8}, "themeVariables": {"fontSize": "15px"}}}%%
flowchart TB
    T1["<b>Task 1 · Assign tags</b><br/>Resource group CST8912 · Canada Central<br/>Tag Cost Center : 000"]
    subgraph T2["Task 2 · Enforce tagging"]
        direction LR
        P1["<b>Require Cost Center tag with Default value</b><br/>Policy: Require a tag and its value on resources<br/>Effect: Deny"]
        S1["<b>cst8912lab8hry1</b><br/>Storage account without the tag<br/>Validation failed · not created"]
        P1 ==>|"create, 3:10 PM"| S1
    end
    subgraph T3["Task 3 · Apply tagging"]
        direction LR
        P2["<b>Inherit the Cost Center tag and its value 000</b><br/>Policy: Inherit a tag from the resource group if missing<br/>Effect: Modify · remediation task · managed identity"]
        S2["<b>cst8912lab8hry2</b><br/>Storage account without the tag<br/>Created · Cost Center : 000 added by the policy"]
        P2 ==>|"create, 3:33 PM"| S2
    end
    subgraph T4["Task 4 · Resource locks"]
        direction LR
        LK["<b>rg-lock</b><br/>Lock on the resource group<br/>Lock type: Delete"]
        DEL["<b>Delete resource group</b><br/>Failed: locks applied<br/>Storage account still there"]
        LK ==>|"try to delete"| DEL
    end
    T5["<b>Task 5 · Clean up the resources</b><br/>Delete the lock, the policy assignment,<br/>the tag and the storage account"]
    T1 ==>|"assign the policy, 3:04 PM"| T2
    T2 ==>|"delete the assignment, assign a new policy, 3:20 PM"| T3
    T3 ==>|"add a lock"| T4
    T4 ==>|"the lock also blocks the delete of the assignment"| T5
    classDef deny fill:#FDECEA,stroke:#C92A2A,stroke-width:2px,color:#4A0E0E
    classDef modify fill:#E5F6EC,stroke:#2B8A3E,stroke-width:2px,color:#0E3B1B
    classDef lock fill:#F3E8FF,stroke:#6F42C1,stroke-width:2px,color:#2E1A47
    classDef plain fill:#E3F0FC,stroke:#0078D4,stroke-width:2px,color:#0B2A4A
    classDef outside fill:#FFFFFF,stroke:#555555,stroke-width:2px,color:#222222
    class P1,S1 deny
    class P2,S2 modify
    class LK,DEL lock
    class T1 plain
    class T5 outside
    style T2 fill:#FFF8F7,stroke:#C92A2A,stroke-dasharray:4 3,color:#4A0E0E
    style T3 fill:#F6FCF8,stroke:#2B8A3E,stroke-dasharray:4 3,color:#0E3B1B
    style T4 fill:#FBF7FF,stroke:#6F42C1,stroke-dasharray:4 3,color:#2E1A47
```

The red box is the Deny policy (Task 2). The green box is the Modify policy (Task 3). The purple box is the lock (Task 4). Everything is in the resource group `CST8912` in Canada Central.

**Where I set each part.**

| Task | Portal page | What I set |
|---|---|---|
| 1 | Resource groups → Create | Resource group `CST8912`, tag `Cost Center : 000` |
| 2 | Policy → Definitions → Assign policy | "Require a tag and its value on resources" on `CST8912` |
| 2 | Storage accounts → Create | `cst8912lab8hry1` without the tag (validation failed) |
| 3 | Policy → Assignments → Assign policy | "Inherit a tag from the resource group if missing" on `CST8912`, with a remediation task |
| 3 | Storage accounts → Create | `cst8912lab8hry2` without the tag (created, tag added) |
| 4 | Resource group → Locks | `rg-lock`, type Delete |
| 5 | Locks, Policy → Assignments, Storage account | Deleted the lock, the assignment, the tag and the storage account |

## Changes from the instructions

| Instruction | What I did | Why |
|---|---|---|
| Storage account table: only Resource group and Storage account name | I also set Region to Canada Central and Primary service to "Azure Blob Storage or Azure Data Lake Storage" | My subscription allows only five regions (I checked in Cloud Shell). Canada Central is the region of the resource group. Primary service is now a required field in the portal. |
| Policy enforcement: Enabled | Default (enforcement) | The portal now shows "Default (enforcement)" and "Do not enforce". The first one means enabled. |
| "Click Next twice and set Parameters" | Clicked Next once | The tabs are now Basics, Parameters, Remediation, Managed identity, Non-compliance messages, Review + create. |
| Remediation tab: "Leave the Create a Managed Identity checkbox unchecked" | Left it unchecked on the Managed identity tab | The checkbox is now on its own tab. |
| Task 4: type the name `cst8912` | Typed `CST8912` | This is the name of the resource group I created in Task 1. |
| Task 5: delete the "Role:Infra" tag | Deleted the `Cost Center : 000` tag | This lab does not have a Role:Infra tag. The storage account had only the Cost Center tag. |
| Task 5: delete the policy assignment (no step about the lock) | Deleted `rg-lock` first | The Delete lock from Task 4 also blocked the delete of the policy assignment. |
| Not in the instructions | Deleted the empty resource group at the end | Nothing was left in it after Task 5. |

## Before Task 1. Check what my subscription can use

- I ran read-only commands in Cloud Shell before I created anything.
- My subscription has the policy "Allowed resource deployment regions". The allowed regions are canadacentral, mexicocentral, northcentralus, westus2 and westus3.
- So I used Canada Central for the resource group and for the storage accounts.

![Policy assignments and allowed regions in Cloud Shell](screenshots/00-precheck-cloudshell.png)

## Task 1. Assign tags via the Azure portal

**Resource groups → Create → Basics.**

| Setting | Value |
|---|---|
| Subscription | Azure for Students |
| Resource group name | `CST8912` |
| Region | Canada Central |

![Create a resource group, Basics](screenshots/01a-rg-basics.png)

**Next: Tags → create a new tag.**

| Setting | Value |
|---|---|
| Name | `Cost Center` |
| Value | `000` |

![Create a resource group, Tags](screenshots/01b-rg-tags.png)

**Review + create → Create.**

![Create a resource group, Review + create](screenshots/01c-rg-review-create.png)

- The resource group `CST8912` has the tag `Cost Center : 000`.

![Tags of the resource group](screenshots/01d-rg-created-tag.png)

## Task 2. Enforce tagging via an Azure policy

**Policy → Authoring → Definitions → search "require a tag".**
- The search shows four built-in definitions. I used the first one, "Require a tag and its value on resources".

![Policy definitions search](screenshots/02a-policy-definitions-search.png)

**Review the definition.**
- Available effects: Deny. Mode: Indexed. It has two parameters, `tagName` and `tagValue`.
- The description says: "Enforces a required tag and its value. Does not apply to resource groups."

![Policy definition](screenshots/02b-policy-definition-detail.png)

**Assign policy → Scope and Basics.**

| Setting | Value |
|---|---|
| Scope | Azure for Students / `CST8912` |
| Assignment name | `Require Cost Center tag with Default value` |
| Description | `Require Cost Center tag with default value for all resources in the resource group` |
| Policy enforcement | Default (enforcement) |

![Assign policy, Basics](screenshots/02d-assign-basics.png)

**Parameters.**

| Setting | Value |
|---|---|
| Tag Name | `Cost Center` |
| Tag Value | `000` |

![Assign policy, Parameters](screenshots/02e-assign-parameters.png)

**Managed identity: the checkbox stays unchecked.**
- The page says this policy does not contain any role definitions.

![Assign policy, Managed identity unchecked](screenshots/02f-assign-remediation-unchecked.png)

**Review + create → Create.**

![Assign policy, Review + create](screenshots/02f2-assign-review-create.png)

- At 3:04 PM the assignment was in the list with the scope `Azure for Students/CST8912`.

![Policy assignments](screenshots/02g-assignment-list.png)

**Storage accounts → Create → Basics, without the tag.**

| Setting | Value |
|---|---|
| Resource group | `CST8912` |
| Storage account name | `cst8912lab8hry1` |
| Region | Canada Central |
| Primary service | Azure Blob Storage or Azure Data Lake Storage |
| Other fields | default values |

![Create a storage account, Basics](screenshots/02h-storage-basics.png)

**Review and create.**
- At 3:10 PM I got "Validation failed". The Create button could not be clicked.

![Validation failed](screenshots/02i-storage-validation-failed.png)

**View error details.**
- The error says: "Resource 'cst8912lab8hry1' was disallowed by policy. (Code: RequestDisallowedByPolicy, Policy(s): Require Cost Center tag with Default value)".
- The storage account was not created.

![Disallowed by policy](screenshots/02j-storage-policy-error-detail.png)

## Task 3. Apply tagging via an Azure policy

**Policy → Assignments → Delete assignment.**
- I deleted the assignment `Require Cost Center tag with Default value`.

![Delete the first assignment](screenshots/03a-assignment-delete.png)

**Assign policy → Scope, Policy definition and Basics.**

| Setting | Value |
|---|---|
| Scope | Azure for Students / `CST8912` |
| Policy definition | Inherit a tag from the resource group if missing |
| Assignment name | `Inherit the Cost Center tag and its value 000 from the resource group if missing` |
| Description | `Inherit the Cost Center tag and its value 000 from the resource group if missing` |
| Policy enforcement | Default (enforcement) |

![Assign the second policy, Basics](screenshots/03b-assign-basics.png)

**Parameters.**

| Setting | Value |
|---|---|
| Tag Name | `Cost Center` |

![Assign the second policy, Parameters](screenshots/03c-assign-parameters.png)

**Remediation.**

| Setting | Value |
|---|---|
| Create a remediation task | checked |
| Policy to remediate | Inherit a tag from the resource group if missing |

- The page says the assignment takes effect on newly created resources. Existing resources are updated by a remediation task.

![Policy remediation page](screenshots/03d-assign-remediation.png)

**Managed identity.**
- "Create a Managed Identity" is checked and cannot be changed. The type is system assigned, the location is Canada Central, and the permission is Contributor.

![Managed identity for the second policy](screenshots/03e-assign-managed-identity.png)

**Review + create → Create.**

![Assign the second policy, Review + create](screenshots/03e2-assign-review-create.png)

- At 3:20 PM the new assignment was in the list. The first assignment was gone.

![Policy assignments after the change](screenshots/03f-assignment-list.png)

**Storage accounts → Create → Basics, without the tag.**

| Setting | Value |
|---|---|
| Resource group | `CST8912` |
| Storage account name | `cst8912lab8hry2` |
| Region | Canada Central |
| Primary service | Azure Blob Storage or Azure Data Lake Storage |
| Other fields | default values |

![Create the second storage account, Basics](screenshots/03g-storage-basics.png)

**Review and create.**
- This time the validation passed and the Create button could be clicked.
- The note in the instructions says the policy can take 5 to 10 minutes. I waited until 3:33 PM and then clicked Create.

![Validation passed](screenshots/03h-storage-validation-passed.png)

![Deployment complete](screenshots/03i-storage-deployment-complete.png)

**Go to resource.**
- The storage account has the tag `Cost Center : 000`. I did not enter this tag.

![Tag on the storage account overview](screenshots/03j-storage-overview-tag-inherited.png)

**Tags blade.**

![Tags blade of the storage account](screenshots/03j2-storage-tags-blade.png)

## Task 4. Configure and test resource locks

**Resource group `CST8912` → Settings → Locks → Add.**

| Setting | Value |
|---|---|
| Lock name | `rg-lock` |
| Lock type | Delete |

![Add lock](screenshots/04a-lock-add.png)

![Lock list](screenshots/04b-lock-list.png)

**Overview → Delete resource group.**
- I entered the name `CST8912`. The page says the resource group and all its dependent resources will be permanently deleted.

![Delete a resource group](screenshots/04c-rg-delete-confirm.png)

**Delete.**
- The notification says: "Failed to delete resource group CST8912. Resource group CST8912 has locks applied. Remove the locks before deleting."
- The storage account was still in the resource group.

![Delete denied by the lock](screenshots/04d-rg-delete-denied.png)

## Task 5. Clean up the resources

**Policy → Assignments → Delete assignment.**
- The delete failed. The message says the scope cannot perform the delete operation because the resource group is locked.

![Delete of the assignment blocked by the lock](screenshots/05a-assignment-delete-blocked-by-lock.png)

**Resource group → Locks → delete `rg-lock`.**

![Lock deleted](screenshots/05b-lock-deleted.png)

**Policy → Assignments → Delete assignment, again.**
- The assignment was deleted. The message says the change takes around 30 minutes to take effect.

![Assignment deleted](screenshots/05c-assignment-deleted.png)

**Storage account → Tags → delete the tag → Apply.**
- I deleted the tag `Cost Center : 000` with the trash can.

![Tag marked to be deleted](screenshots/05d-storage-tag-delete.png)

![Tag deleted](screenshots/05d2-storage-tag-deleted.png)

**Overview → Delete.**
- I entered the name `cst8912lab8hry2` and deleted the storage account.

![Delete storage account](screenshots/05e-storage-delete-confirm.png)

![No storage accounts](screenshots/05f-storage-deleted.png)

**The empty resource group.**
- After Task 5 the resource group `CST8912` had no resources. I also deleted it.

![Empty resource group after Task 5](screenshots/05g-rg-empty-after-task5.png)

![Delete the empty resource group](screenshots/05g2-rg-delete-confirm.png)

- Only `NetworkWatcherRG` is left. It was in my subscription before this lab.

![Resource groups after the clean up](screenshots/05h-rg-deleted.png)

## Findings

- The Deny policy stopped the storage account before it was created. The check happened on the Review page, and the error showed the name of my policy assignment.
- With the Modify policy, the same storage account settings worked, and the tag `Cost Center : 000` was added. The value came from the tag of the resource group. A resource does not get the tag of its resource group without this policy.
- The two policies were different on the Managed identity tab. The Deny policy needed no identity. The Modify policy needed a managed identity with the Contributor role, because it changes resources.
- The Delete lock stopped me from deleting the resource group, and I am the Owner. It also stopped the delete of the policy assignment on that resource group. I had to delete the lock first.
- A policy needs a few minutes to work. The Deny policy was working 6 minutes after I assigned it. For the Modify policy I waited 13 minutes before I created the storage account.
- In the lecture, tags are used to track cost by cost center. This lab shows how a policy can keep these tags on the resources.

## Clean up

- Deleted in this order: lock `rg-lock`, policy assignment, tag on the storage account, storage account `cst8912lab8hry2`, resource group `CST8912`.
- The first policy assignment was deleted in Task 3. The storage account `cst8912lab8hry1` was never created.
- Nothing from this lab is left in my subscription.
