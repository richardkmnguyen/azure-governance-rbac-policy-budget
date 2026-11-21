Watch me explain this lab: [LoomLink](PASTE_LOOM_LINK_HERE)

# azure-governance-rbac-policy-budget

Built three layers of governance around one Azure resource group and **tested each one instead of just configuring it**. A new **Junior Developer** account in Entra ID got the **Reader** role and was blocked with `AuthorizationFailed` when it tried to create a storage account. An **Azure Policy** Deny rule blocked a `Standard_D2als_v7` VM, even for my Owner account, while still allowing the approved `Standard_F1als_v7`. A **$50 budget** now emails me at 80% of actual spend and when spend is forecast to pass 100%. Each control maps to a function of the **NIST Cybersecurity Framework**.

# Azure Cloud Series, Lab 05: Governance and Security Hardening

**Microsoft Entra ID · Azure RBAC · Azure Policy · Cost Management + Budgets · NIST CSF · Azure Portal**

The first four labs were about building things. This one is about controlling who can build, what they can build, and how much it's allowed to cost. Those are the guardrails every real Azure environment needs before more people get access. This is the fifth lab of a six week Azure series that ends in a capstone project.

| | |
|---|---|
| **Platform** | Azure Portal against my personal Azure subscription, East US. An incognito browser window to sign in as the test user |
| **Time** | About 1.5 hours, including waiting for the policy to take effect and working around VM sizes my subscription doesn't offer |
| **Cost** | $0. No VM was ever deployed; resource groups, role assignments, policies and budgets are free |

---

## What this lab is for

The business problem: if everyone has broad access and there are no guardrails, someone eventually deletes the wrong thing, deploys a VM that's far bigger than needed, or leaves something running that nobody notices until the bill arrives.

Governance answers three questions, and each one has its own Azure tool:

- **Who can act? (RBAC).** Roles bundle permissions and get assigned to people at a scope. New users start with zero access (deny by default), so everything has to be granted on purpose.
- **What can be deployed? (Azure Policy).** Rules apply to configurations no matter who deploys them. A Deny policy blocks even an Owner, so governance can't be bypassed by raising your own permissions.
- **When is spend drifting? (Budgets).** Alerts fire at thresholds. A budget **does not** stop, resize or delete anything. It's a smoke detector, not a sprinkler.

| NIST CSF function | Where it shows up in this lab |
|---|---|
| **Identify** | Creating the Junior Developer identity in Entra ID before granting anything |
| **Protect** | The Reader role (RBAC) and the VM size Deny policy (Azure Policy) |
| **Detect** | Budget alerts: a spend spike can mean a forgotten resource, a runaway process or a compromised account |
| **Respond** | Getting the alert email and investigating in Cost analysis |
| **Recover** | Cleanup: deleting the test user and resource group to get back to a known good state |

---

## Architecture

```
Admin (my account, Owner on the subscription)
        │
        ├── Microsoft Entra ID
        │       └── Junior Developer (junior-dev-richardnguyen@<tenant>.onmicrosoft.com)
        │
        └── rg-lab05-gov-richardnguyen (East US)  ◄── scope for every control
                │
                ├── RBAC:    Junior Developer = Reader            (can view, can't create)
                ├── Policy:  Restrict-VM-Sizes (Deny)             (only F1als_v7, F1alds_v7)
                └── Budget:  Monthly-Lab-Budget, $50/month        (80% actual, 100% forecast → email)

Junior Developer (incognito)  ──►  sees only this resource group
                              ──►  + Create storage account  ──►  AuthorizationFailed
Admin                         ──►  D2als_v7 VM  ──►  blocked by Restrict-VM-Sizes
Admin                         ──►  F1als_v7 VM  ──►  passes validation (never deployed)
```

<!-- Optional: drag an architecture diagram image here -->

| Component | Role |
|---|---|
| **`rg-lab05-gov-richardnguyen`** | The scope for all three controls, so nothing else in the subscription is affected |
| **Junior Developer** | Test identity in Entra ID that represents a new hire |
| **Reader role assignment** | View-only access, scoped to this one resource group |
| **`Restrict-VM-Sizes`** | Assignment of the built-in *Allowed virtual machine size SKUs* policy, effect Deny |
| **`Monthly-Lab-Budget`** | $50 monthly budget with an 80% Actual and a 100% Forecasted email alert |

---

## What I built

### 1. Created the lab resource group
Created `rg-lab05-gov-richardnguyen` in East US. Every role assignment, policy and budget in the lab is scoped to it, so I could practice governance without affecting anything else in my subscription.

<img width="1466" height="164" alt="GovernanceRGCreated" src="https://github.com/user-attachments/assets/8d2b0829-42ff-41eb-8abc-7f06b5908e68" />

### 2. Created a user and assigned Reader (Identify + Protect)
In **Microsoft Entra ID → Users**, I created **Junior Developer** (`junior-dev-richardnguyen`) with a password I set myself. Creating the user grants nothing. Azure starts every identity with zero permissions.

Then in the resource group's **Access control (IAM)**, I added a role assignment: **Reader → Junior Developer**, scoped to this resource group only. Under **Role assignments**, Junior Developer appears as Reader with scope *This resource*, not inherited from the subscription.

<img width="1133" height="190" alt="GovernanceEntraUserCreated" src="https://github.com/user-attachments/assets/f370fa6a-9e28-4822-9a7c-7179d71afbd2" />
<img width="650" height="272" alt="GovernanceIAMReviewAssign" src="https://github.com/user-attachments/assets/574de5ad-d093-4ce6-88aa-f0a662720c52" />
<img width="1597" height="438" alt="GovernanceIAMRoleAssignments" src="https://github.com/user-attachments/assets/a1398699-e68b-4885-aec7-de0e613e942b" />


### 3. Tested the Reader role as Junior Developer
In an incognito window I signed in as Junior Developer. Azure made me register **Microsoft Authenticator** first, because MFA is now required for every portal sign-in. Under **Resource groups**, only `rg-lab05-gov-richardnguyen` was visible.

Inside the resource group, **+ Create** opened the Marketplace like it does for any account. Reader can browse; it just can't write. I picked **Storage account**, filled in the form, and the create was blocked:

```
AuthorizationFailed: The client 'junior-dev-richardnguyen@…' does not have
authorization to perform action 'Microsoft.Storage/storageAccounts/write'
```

<img width="1920" height="279" alt="GovernanceJuniorRGView" src="https://github.com/user-attachments/assets/cccb94bd-b757-4be8-9b48-5f7151ff145d" />
<img width="1919" height="326" alt="GovernanceJuniorAuthorizationFailed" src="https://github.com/user-attachments/assets/eacee01a-6e8b-47a4-9763-205f17f7e898" />

> Least privilege in practice: a new hire can see the environment and learn it, but can't change anything in it.

### 4. Assigned the VM size policy (Protect)
In **Policy → Assignments → Assign policy**, I assigned the built-in **Allowed virtual machine size SKUs** definition to the resource group as `Restrict-VM-Sizes`:

| Setting | Value |
|---|---|
| Scope | `rg-lab05-gov-richardnguyen` |
| Policy enforcement | **Default (enforcement)** |
| Allowed Size SKUs | `Standard_F1als_v7`, `Standard_F1alds_v7` (1 vCPU, 2 GiB each) |
| Effect | Deny (fixed by the built-in definition) |

The **definition** is Microsoft's prewritten rule. The **assignment** activates it at a scope; a definition does nothing until it's assigned. The **parameter** list is what makes it specific to this environment.

<img width="586" height="697" alt="GovernancePolicyParameters" src="https://github.com/user-attachments/assets/597523d3-6e32-4a03-b253-23ff2d262bf7" />
<img width="1895" height="295" alt="GovernancePolicyAssignment" src="https://github.com/user-attachments/assets/2f7dfb48-21ce-42b8-a8cf-61a76db743cd" />

### 5. Tested the policy in both directions
After waiting for the assignment to take effect (it can take 10 to 30 minutes), I started a VM in the resource group with my **Owner** account:

- **`Standard_D2als_v7` (not allowed):** validation failed with a policy error naming `Restrict-VM-Sizes`. My own Owner account was blocked.
- **`Standard_F1als_v7` (allowed):** Review + create loaded with no policy error and the Create button enabled. That's a pass. I closed the blade without deploying anything.

<img width="1910" height="951" alt="GovernancePolicyDeniedD2alsFinal" src="https://github.com/user-attachments/assets/91bd3b4e-a0b2-443b-a5ad-d1e15ca6fe03" />
<img width="1909" height="948" alt="GovernancePolicyAllowedF1als" src="https://github.com/user-attachments/assets/c6ece34b-a5b9-4629-bc31-11740c633738" />

> Testing both directions matters: a policy that blocks everything is just as broken as one that blocks nothing.

### 6. Created a budget with two alerts (Detect)
In the resource group's **Cost Management → Budgets**, I created `Monthly-Lab-Budget`:

| Setting | Value |
|---|---|
| Amount | $50, resets every billing month |
| Expiration | One year out |
| Alert 1 | **Actual**, 80%: fires once $40 has actually been spent |
| Alert 2 | **Forecasted**, 100%: fires when Azure projects spend will pass $50 by month end |
| Recipient | My email |

Actual tells me what has already happened. Forecasted warns me while there's still time to act. I didn't add an Action Group. In production, one could post to a channel, open a ticket or trigger automation that shuts down non-critical resources.

<img width="1409" height="637" alt="GovernanceBudgetAlerts" src="https://github.com/user-attachments/assets/64b22cf1-967a-48a6-b267-0987ba55b63d" />
<img width="1901" height="202" alt="GovernanceBudgetCreated" src="https://github.com/user-attachments/assets/d6549ce5-bcf1-4821-94c6-6cd263bceca8" />

### 7. Cleaned up (Recover)
Deleted the resource group, which also removes the role assignment and policy assignment scoped to it. Then I deleted Junior Developer from Entra ID and confirmed the budget and policy assignment were gone.

<!-- cleanup-verified.png here -->

---

## Changes from the original guide I planned to follow

| Original guide | What I did instead | Why |
|---|---|---|
| Allowed sizes `Standard_B1s` and `Standard_B1ms` | `Standard_F1als_v7` and `Standard_F1alds_v7` | My subscription has no B-series sizes at all. B-series v1 is retiring in November 2028 and is already restricted for new deployments. The F1 v7 sizes are my smallest available (1 vCPU, 2 GiB), so they fill the same "cheap and approved" role |
| Denied test size `Standard_D2s_v3` | `Standard_D2als_v7` | D2s_v3 isn't available on my subscription. Testing with a size I know I can deploy proves the block came from the policy, not from availability |
| Policy enforcement "Enabled" | **Default (enforcement)** | Current name in the portal. The other options are *Do not enforce* (audit only) and *Enroll* (a preview mode that only enforces on scopes you enroll separately) |
| Expect a green "Validation passed" banner | Treated "no policy error + Create enabled" as a pass | The current VM create flow often skips the banner |
| — | Planned the budget setup during the policy propagation wait | Turns 10 to 30 minutes of waiting into useful work |
| — | Checked that the budget and policy assignment were gone after cleanup | Confirms nothing outlived the resource group |

---

## Common issues

| Symptom | Cause | Fix |
|---|---|---|
| MFA registration prompt on the test user's first sign-in | Azure requires MFA for every portal sign-in | Register Microsoft Authenticator. This is expected, and it's NIST Protect in action |
| **+ Create** as Junior Developer opens the Marketplace | That's how create works for every account; Reader can browse | Pick a resource, fill in the form, and the block appears at the resource group or at Review + create |
| Ticked `Standard_F1` in the Allowed Size SKUs list | `Standard_F1` is its own (older, F-series v1) size, not a family that covers `F1als_v7` | Edit the assignment, search **F1als** and tick the exact v7 sizes, untick `Standard_F1`. Policy matches exact size names |
| Disallowed size passes validation | The assignment hasn't propagated yet | Wait 15 minutes and retry; editing an assignment can restart the wait |
| Sizes from the guide missing from the size picker | Regional or subscription restrictions | Use the smallest sizes your subscription offers and keep the same allowed vs denied logic |

---

## Key takeaways

1. **RBAC is about who, Policy is about what.** Reader stopped a person; the Deny policy stopped a configuration, even from an Owner. A real governance model needs both.
2. **Deny by default.** Creating a user grants nothing. Every permission is a deliberate choice at a deliberate scope.
3. **Scope is part of the control.** Everything here was scoped to one resource group, so the Junior Developer couldn't even see the rest of the subscription.
4. **Precision matters in policy parameters.** `Standard_F1` and `Standard_F1als_v7` look close, but the policy compares exact names. Picking the wrong one would have blocked my own approved size.
5. **A budget detects; it doesn't protect.** It emails me. Stopping the spend is still my job, or the job of automation I attach to it.
6. **Test controls in both directions.** Blocked when it should be, allowed when it should be. A control I haven't tested is a control I can't trust.

If I did this again, I'd deploy the whole setup with Terraform (`azurerm_role_assignment`, `azurerm_resource_group_policy_assignment`, `azurerm_consumption_budget_resource_group`) so the guardrails are code, like Lab 04. I'd also verify the policy parameters from the CLI right after assigning, and try a policy in *Do not enforce* mode first to see what it would block before turning it on.

---

## Security notes

This is a lab configuration. Some of it is solid and some of it would not be acceptable in production:

**What the lab does well**
- **Least privilege at the narrowest scope.** Reader on one resource group, not the subscription.
- **Deny by default verified.** The test user was confirmed to have no access until the role was assigned, and no write access after.
- **MFA enforced** for the test user's first sign-in.
- **Guardrails that apply to admins too.** The Deny policy blocked my Owner account.
- **Spend monitoring** with both an actual and a forecasted threshold.
- **Personal subscription, everything deleted** after the lab, including the test identity, so it doesn't linger as unused attack surface.

**What I would change for production**
- **Assign roles to groups, not individuals,** so access follows team membership and is easier to review.
- **Custom roles** with exactly the permissions a job needs, instead of only built-in roles.
- **Privileged Identity Management (PIM)** for just-in-time, time-limited elevation to Contributor or Owner.
- **Policy initiatives at the management group level** (allowed locations, required tags, allowed SKUs), rolled out in *Do not enforce* mode first, then switched to enforcement.
- **Action Groups on budgets** that notify a team channel or ticket queue, not one person's inbox.
- **Regular access reviews** in Entra ID so stale assignments get removed.
- **Governance as code** (Terraform or Bicep) so roles, policies and budgets are version-controlled and reviewed.

---

## Verification

| Check | How I verified it |
|---|---|
| Test user exists | Entra ID → Users: Junior Developer with UPN `junior-dev-richardnguyen@…` |
| Reader at the right scope | Resource group → IAM → Role assignments: Junior Developer, Reader, *This resource* |
| Visibility limited | As Junior Developer, Resource groups showed only `rg-lab05-gov-richardnguyen` |
| Write blocked | Storage account create as Junior Developer: `AuthorizationFailed` |
| Policy assigned | Policy → Assignments: `Restrict-VM-Sizes` scoped to the resource group |
| Disallowed size blocked | `Standard_D2als_v7` VM: validation failed, policy `Restrict-VM-Sizes` |
| Allowed size permitted | `Standard_F1als_v7` VM: no policy error, Create enabled (not deployed) |
| Budget configured | Budgets: `Monthly-Lab-Budget`, $50, 80% Actual + 100% Forecasted, my email |
| Nothing left behind | Resource group, test user, budget and policy assignment all deleted |

---

## Repo contents

| File | Purpose |
|---|---|
| `README.md` | This write-up |
| `screenshots/` | Lab screenshots, numbered in the order they were taken |

Everything in this lab was done in the Azure Portal, so there's no code to run. The steps above are enough to rebuild it in any subscription. Swap in the smallest VM sizes your subscription offers.

---

## Teardown

1. Resource group → **Delete resource group** (also removes the role and policy assignments scoped to it)
2. Entra ID → Users → delete **Junior Developer**, then permanently delete it from **Deleted users**
3. Cost Management → Budgets (subscription scope): confirm `Monthly-Lab-Budget` is gone
4. Policy → Assignments: confirm `Restrict-VM-Sizes` is gone

---

## Next steps

- Rebuild the role assignment, policy and budget in **Terraform**, carrying on from Lab 04
- Add a **policy initiative** with allowed locations and required tags (for example `CostCenter`)
- Attach an **Action Group** to the budget that triggers automation, not just an email
- Try **PIM** for just-in-time elevation and an **access review** on the Reader assignment
- Carry these guardrails into the series capstone so it's governed from day one

---

**Status: complete.** Created the resource group → created Junior Developer and assigned Reader at resource group scope → confirmed `AuthorizationFailed` as that user → assigned `Restrict-VM-Sizes` (F1als_v7, F1alds_v7, Deny) → D2als_v7 blocked, F1als_v7 allowed → created a $50 budget with 80% Actual and 100% Forecasted alerts → cleaned everything up.
