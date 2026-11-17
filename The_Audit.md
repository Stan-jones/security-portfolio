# Privilege Audit

| Method               | Sees                                        | Misses                              |
|----------------------|---------------------------------------------|-------------------------------------|
| IAM blade / export   | Active assignments at scope, inherited      | Group members, orphaned principals  |
| Azure CLI            | Same, plus null principalName (orphans)     | One scope per run                   |
| Resource Graph (KQL) | Whole tenant in one query                   | Eligible assignments                |
| PIM export           | Eligible vs active, activation history      | Assignments outside PIM             |

## Stage 1: Access Control (IAM) Role Assignments Export

**Method:** The Access Control (IAM) blade is the no-tooling starting point for any Azure RBAC audit. Every scope (management group, subscription, resource group, resource) has its own IAM page. Because clicking through scopes one at a time doesn't scale, auditors use **Download role assignments** at the subscription level, the highest scope that supports the "Children" option, to export every assignment, including inherited grants and those on child resources.

**What I did:** My Reader-level operative account couldn't pull the full export, so I reviewed the provided `Role_Assignments.csv` and looked for over-privileged principals.

**Finding:** One account held the **Owner** role across multiple scopes, far more standing privilege than any single identity should have.

> <img width="505" height="529" alt="image" src="https://github.com/user-attachments/assets/d0e8cf9a-660c-44f2-9f3c-547a0e6e9571" />

**Limitation:** The export lists groups but not their members, and it may omit orphaned ("Identity not found") assignments.

---

## Stage 2: Azure CLI

**Method:** The Azure CLI (or PowerShell's `Get-AzRoleAssignment` enumerates the same assignments as the portal, but it also exposes entries the portal buries, such as deleted accounts.

**Command run (Azure Cloud Shell):**

```bash
az role assignment list --resource-group "<resource group>"
```

**Finding:** The output revealed an **orphaned role assignment** (right)

> <img width="1615" height="437" alt="image" src="https://github.com/user-attachments/assets/23917a43-239c-4a03-9b22-63685073093c" />


*Left: screenshot from my Azure output. Right: JSON file provided (because my operator account is scoped too narrowly)*

**Why orphaned accounts matter:** When a user, group, or service principal is deleted, its role assignments can remain, showing as "Identity not found" or "Unknown." They add clutter, hide real access, and create risk, since a recycled or recreated principal ID could silently inherit the old permissions. They should be identified and removed during every access review.

---

## Stage 3: Azure Resource Graph

**Method:** The IAM blade and CLI work one scope at a time. **Azure Resource Graph** uses KQL (Kusto Query Language) to query the `authorizationresources` table across every subscription in the tenant in one execution.

**Why KQL matters:** It's a fast, read-only language for sweeping an entire tenant for risky access, such as orphaned principals, redundant assignments across scopes, privileged roles, or unused custom roles.

**Query used** (principal ID redacted):

```kql
authorizationresources
| where type =~ 'microsoft.authorization/roleassignments'
| extend principalId = tostring(properties.principalId)
| extend description = properties.description
| where principalId == '<ORPHANED_PRINCIPAL_ID>'
| project name, principalId, principalType = properties.principalType, scope = properties.scope, description
```
<img width="2524" height="1251" alt="image" src="https://github.com/user-attachments/assets/f6869bf2-a281-4407-bb39-64ceb8cb5926" />


**What I did:** I ran the query with the orphaned account's principal ID from Stage 2. It returned **no results**, because my operative account lacks the permissions to see those assignments. However, I could remove the `where principalId == ...` filter to broaden the query to all visible assignments (screenshot above).

**Takeaway:** An empty result isn't proof of "no access." It can mean the auditing account lacks visibility, so auditors must confirm their *own* permissions before trusting a search.

## Stage 4: Privileged Identity Management (PIM)

**Method:** Stages 1-3 only show *active* assignments. PIM shows who holds privilege versus who must *activate* it.

**What I did:** My operative account couldn't open the subscription-level PIM view, but selecting the subscription together with a resource group I had access to load the management view and it allowed the export.

**Finding:** The export showed account names that were hidden in earlier stages, including a stale account whose name indicated it was marked for deletion [redacted for flag] but still held a role assignment.

> <img width="352" height="177" alt="image" src="https://github.com/user-attachments/assets/b493e040-96ff-4f84-b3ae-5284dcfc5be0" />


**Why it matters:** Eligible assignments don't appear in the IAM, CLI, or Resource Graph methods, so an audit without PIM can miss privilege that an account can activate on demand. Stale accounts like this should have their assignments removed and be deleted.

## Stage 5: The Hunt

**Objective:** Find the over-provisioned account holding Owner on a hidden resource group.

<details>
<summary>⚠️ Spoiler: Stage 5 walkthrough (click to expand)</summary>

**What I did:**
1. In PIM → **My roles**, I found an eligible assignment for the `hidden-pim` resource group and **activated** it. Eligible roles grant nothing until activated, so this is how I got access.
2. I opened the `hidden-pim` resource group in Resource Manager → **Access control (IAM)** → **Download role assignments**.
3. I reviewed the export and found an account with **Owner** with a suspicious condition attached.

</details>

**Finding:** The Owner assignment carried a condition restricting role assignment writes and deletes for three privileged roles: Owner, User Access Administrator, and RBAC Administrator.

> <img width="1140" height="96" alt="image" src="https://github.com/user-attachments/assets/62c3b52f-6b1b-4568-aa98-e4e1edc8289e" />


**Why it matters:** A hidden resource group with a standing Owner is hard to find, and conditions are easy to overlook because the portal doesn't show them upfront. Auditors should read the Condition column, not just the role name.

---

## Key Findings and Recommendations

| Stage | Finding | Recommendation |
|---|---|---|
| 1. IAM | One account held Owner at multiple scopes | Apply least privilege; replace standing Owner with narrower roles |
| 2. CLI | Orphaned role assignment ("Identity not found") | Remove orphaned accounts; automate the check on a schedule |
| 3. Resource Graph | Query returned nothing without sufficient permissions | Confirm auditor visibility before trusting results; use tenant-wide searches if needed |
| 4. PIM | Stale "delete" account still held an assignment | Remove accounts flagged for deletion; review Eligible, not only Active |
| 5. Hunt | Hidden resource group with a conditioned Owner | Inventory all resource groups; make Owner PIM-eligible with approval; document conditions |

**Overall takeaway:** No single method sees everything. IAM exports miss group members and orphans, the CLI and Resource Graph see only active assignments, and PIM covers eligibility. A complete RBAC audit combines all of them.
