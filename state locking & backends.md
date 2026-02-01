# 1️⃣ Terraform state locking & backends (deep dive)

## What Terraform *state* really is

Terraform state answers one question:

> “What resources exist, and how do I talk to them again?”

State contains:

* Resource IDs (instance IDs, ARNs, URLs)
* Provider-specific metadata
* Dependency graph
* Output values

Terraform **cannot safely operate without state**.

---

## Why state locking exists

Terraform assumes:

* **One writer at a time**
* **State must be consistent**

Without locking:

* Two applies run concurrently
* Both read the same state
* Both make conflicting changes
* Last writer wins → **state corruption**

State locking prevents this by enforcing **mutual exclusion**.

---

## What happens during state locking

When you run `terraform apply` (or `destroy`):

1. Terraform requests a **lock** from the backend
2. Backend grants lock if none exists
3. Lock metadata is written:

   * Who locked it
   * When
   * Operation type
4. Terraform proceeds
5. Lock is released on completion (or crash recovery)

If a lock exists:

```
Error: Error acquiring the state lock
```

---

## Backend types (and how locking works)

### 1. Local backend (default)

```hcl
terraform {
  backend "local" {}
}
```

Locking:

* Uses a local `.terraform.tfstate.lock.info` file
* Works **only on the same filesystem**
* ❌ No protection across machines

✅ Fine for learning
❌ Dangerous for teams

---

### 2. S3 backend (most common in prod)

```hcl
backend "s3" {
  bucket         = "tf-state"
  key            = "prod/terraform.tfstate"
  region         = "us-east-1"
  dynamodb_table = "terraform-locks"
}
```

Locking mechanism:

* **DynamoDB conditional writes**
* One lock item per state file
* Strong consistency

Failure mode:

* If process crashes, lock remains
* Must be manually unlocked

```bash
terraform force-unlock LOCK_ID
```

✅ Industry standard
✅ Safe for teams
✅ Auditable

---

### 3. Terraform Cloud / Enterprise backend

Locking:

* Managed automatically
* Per workspace
* Integrated with runs & approvals

Extras:

* Remote execution
* Policy checks
* UI visibility of locks

✅ Best UX
❌ Vendor lock-in

---

### 4. Other remote backends (GCS, AzureRM, etc.)

All implement:

* Atomic lock acquisition
* Lease-based or conditional writes

Quality varies, but concept is identical.

---

## What happens if locking fails

Scenarios:

* Someone else is applying
* CI job crashed
* Network issue during unlock

Best practice:

* **Investigate before force-unlock**
* Never force-unlock during an active apply

Rule of thumb:

> If you didn’t start the lock, assume it’s dangerous to remove.

---

## Backend ≠ State

Important distinction:

| Concept | Meaning                     |
| ------- | --------------------------- |
| Backend | Where & how state is stored |
| State   | The data itself             |

Changing backend ≠ deleting state
Destroying infra ≠ deleting backend storage

---

# 2️⃣ Safe `terraform apply` patterns for production

This is the part most teams get wrong.

---

## Golden rules for prod applies

### Rule 1: **Plan and apply must be separated**

Never do this in prod:

```bash
terraform apply -auto-approve
```

Instead:

```bash
terraform plan -out=tfplan
terraform apply tfplan
```

Why:

* Guarantees the apply matches the reviewed plan
* Prevents config drift between steps

---

### Rule 2: **Apply only from CI**

Humans should not apply prod infra locally.

Pattern:

* Engineers submit PRs
* CI runs `plan`
* Humans review
* CI applies

This gives:

* Audit trail
* Reproducibility
* Access control

---

### Rule 3: **Use immutable plans**

Plans are **binary snapshots**.

Good:

```bash
terraform plan -out=tfplan
terraform apply tfplan
```

Bad:

```bash
terraform apply
```

Because:

* Variables can change
* State can change
* Code can change

---

### Rule 4: **Lock prod credentials**

Prod IAM should:

* Be usable only by CI
* Have no long-lived user creds
* Require approvals upstream

Terraform is powerful — **limit blast radius**.

---

### Rule 5: **No `-target` in prod**

`-target`:

* Bypasses dependency graph
* Skips safety checks
* Creates hidden drift

Use only for:

* Emergency recovery
* Provider bugs
* One-time fixes

Never as a workflow.

---

## A safe production apply workflow (example)

### 1. Pull request opened

```text
PR opened → CI runs terraform plan
```

Artifacts:

* Saved plan file
* Plan output attached to PR

---

### 2. Human review

Reviewers check:

* Resource count changes
* Replacements (`-/+`)
* Deletes (`-`)
* Provider changes

No review → no apply.

---

### 3. Approval gate

* GitHub approval
* Manual pipeline approval
* Change management hook

---

### 4. Apply stage (CI only)

```bash
terraform apply tfplan
```

State:

* Locked
* Modified once
* Audited

---

## Environment isolation (critical)

Never share state between environments.

Correct:

```
state/dev/terraform.tfstate
state/stage/terraform.tfstate
state/prod/terraform.tfstate
```

Even better:

* Separate accounts
* Separate credentials
* Separate backends

---

## Drift handling in prod

Drift happens.

Safe pattern:

1. Detect drift with `terraform plan`
2. Review *why* it happened
3. Reconcile with `apply`
4. Never “just force it”

---

## Red flags in production Terraform

🚩 `terraform apply -auto-approve`
🚩 Applying from a laptop
🚩 Shared state across envs
🚩 Force-unlocking blindly
🚩 Frequent `-target` usage

If you see these, prod is already unsafe.

---

## TL;DR

### State locking:

* Prevents concurrent writes
* Backend-enforced
* Absolutely mandatory for teams

### Safe prod applies:

* CI-only
* Plan → review → apply
* Immutable plans
* Locked credentials
* No shortcuts

