# `terraform destroy` explained

`terraform destroy` **deletes all infrastructure managed by the current Terraform state**.

It does **not** look at what exists in your cloud account — it destroys **what’s in state**.

Think of it as:

> “Make everything Terraform knows about go away.”

---

## What happens during `terraform destroy`

### 1. Loads configuration

Terraform reads:

* `*.tf` files
* Variables (`.tfvars`, env vars)
* Modules from `.terraform/modules/`

Even though it’s a destroy, **config is still required**.

---

### 2. Loads and locks state

Terraform:

* Fetches the current state from the backend
* Acquires a **state lock** (if supported)

Purpose:

* Prevents concurrent applies/destroys
* Avoids state corruption

---

### 3. Refreshes real infrastructure

Terraform queries provider APIs to:

* Verify resources still exist
* Detect drift
* Get current IDs and metadata

This ensures Terraform knows *what* to destroy.

---

### 4. Builds a destroy plan

Terraform computes a plan where:

* All managed resources are marked for deletion
* Dependencies are reversed (children destroyed before parents)

This plan is shown to you unless auto-approved.

---

### 5. Confirmation prompt

By default, Terraform asks:

```
Do you really want to destroy all resources?
```

Nothing happens until you confirm.

Skip prompt with:

```bash
terraform destroy -auto-approve
```

⚠️ Dangerous in production.

---

### 6. Executes destruction

Terraform:

* Calls provider APIs to delete resources
* Destroys in dependency-safe order
* Handles partial failures

This is **irreversible** at the API level.

---

### 7. Updates (or removes) state

As resources are destroyed:

* They are removed from state
* After completion:

  * State becomes empty (or nearly empty)
  * State file is written back to backend

---

## Files created or modified by `terraform destroy`

### 1. `terraform.tfstate` (or remote state)

After destroy:

* State still exists
* Resource entries are removed
* Outputs are cleared

Terraform does **not** delete the state file itself.

---

### 2. `terraform.tfstate.backup` (local backend)

A backup of the previous state is created automatically.

---

### 3. `.terraform/`

* Providers and modules remain
* Directory is **not cleaned up**

You must delete it manually if needed.

---

## What `terraform destroy` requires

| Requirement       | Needed |
| ----------------- | ------ |
| `terraform init`  | ✅      |
| Valid credentials | ✅      |
| State access      | ✅      |
| Lock access       | ✅      |

---

## Common flags

| Flag                   | What it does               |
| ---------------------- | -------------------------- |
| `-auto-approve`        | Skip confirmation          |
| `-target=resource`     | Destroy specific resources |
| `-refresh=false`       | Skip refresh               |
| `-parallelism=n`       | Control delete concurrency |
| `-var-file=env.tfvars` | Use specific vars          |

⚠️ `-target` during destroy can leave orphaned resources.

---

## Failure scenarios (important)

If destroy fails:

* Some resources may already be deleted
* Some remain in state
* Next `destroy` or `apply` reconciles

Terraform **does not roll back deletions**.

---

## What destroy does *not* do

❌ Delete resources not in state
❌ Clean up manually created infra
❌ Remove backend storage (S3 bucket, workspace, etc.)
❌ Protect production automatically

---

## Typical usage patterns

### Local / dev environments

```bash
terraform destroy
```

### CI cleanup (ephemeral environments)

```bash
terraform destroy -auto-approve
```

### Safe prod pattern

* Restricted IAM
* Manual approval
* No auto-approve
* Often disabled entirely

---

## Summary table

| Command   | Effect on infra | Effect on state |
| --------- | --------------- | --------------- |
| `plan`    | None            | None            |
| `apply`   | Create / update | Writes          |
| `destroy` | Delete          | Removes         |

---

## TL;DR

`terraform destroy`:

* Deletes **everything Terraform manages**
* Is state-driven, not account-driven
* Is irreversible
* Should be tightly controlled

