# `terraform apply` explained

`terraform apply` **executes the changes** required to make real infrastructure match your Terraform configuration.

Unlike `plan`, this **actually talks to cloud APIs and mutates resources**.

Think of it as:

> “Make reality match the code.”

---

## What happens during `terraform apply`

### 1. Loads configuration

Terraform reads:

* `*.tf` files
* Variables (`.tfvars`, env vars)
* Modules from `.terraform/modules/`

---

### 2. Loads and locks state

Terraform:

* Fetches the current state from the backend
* Acquires a **state lock** (if supported)

Purpose:

* Prevents concurrent applies
* Avoids state corruption

---

### 3. Refreshes real infrastructure

Terraform queries provider APIs to get **actual resource state**.

This ensures:

* State reflects reality *before* changes
* Diffs are computed against current infrastructure

Can be skipped with:

```bash
terraform apply -refresh=false
```

---

### 4. Creates an execution plan (internally)

If you didn’t pass a saved plan file:

```bash
terraform apply
```

Terraform:

* Generates a plan
* Shows it
* Asks for confirmation

If you did pass one:

```bash
terraform apply tfplan
```

Terraform:

* Skips planning
* Executes exactly what’s in the plan

---

### 5. Executes resource changes

Terraform performs actions in dependency order:

* **Create** resources (`+`)
* **Update** resources (`~`)
* **Destroy** resources (`-`)
* **Replace** resources (`-/+`)

This is where:

* API calls happen
* Failures can occur
* Partial success is possible

---

### 6. Updates state

After each successful operation:

* Terraform updates the state file
* Writes the new resource attributes
* Persists the final state to the backend

This step is **critical** — state is the source of truth.

---

## Files created or modified by `terraform apply`

### 1. `terraform.tfstate` (or remote state)

| Backend        | Result                      |
| -------------- | --------------------------- |
| Local backend  | `terraform.tfstate` updated |
| Remote backend | State updated remotely      |

Contains:

* Resource IDs
* Metadata
* Provider-specific attributes
* Dependency mappings

🚨 **Never commit this file**

---

### 2. `terraform.tfstate.backup` (local backend)

Created automatically:

* Backup of the previous state
* Used for recovery

---

### 3. `.terraform/` (may be reused)

* Providers and modules already downloaded
* No new files unless something changes

---

### 4. Output values

Outputs are:

* Computed
* Stored in state
* Printed to the terminal

---

## What `terraform apply` requires

| Requirement          | Needed |
| -------------------- | ------ |
| `terraform init`     | ✅      |
| Valid plan           | ✅      |
| Provider credentials | ✅      |
| State lock access    | ✅      |

---

## Common flags

| Flag                | What it does             |
| ------------------- | ------------------------ |
| `-auto-approve`     | Skip confirmation        |
| `-refresh=false`    | Skip refresh             |
| `-target=resource`  | Apply specific resources |
| `-parallelism=n`    | Control concurrency      |
| `-replace=resource` | Force recreation         |

---

## Failure scenarios (important)

Terraform **does not roll back** automatically.

If apply fails:

* Some resources may be created
* State may be partially updated
* Next `plan` reconciles the drift

This is why **idempotency matters**.

---

## What apply does *not* do

❌ Validate correctness of architecture
❌ Guarantee success of all API calls
❌ Automatically roll back failures
❌ Protect you from bad config

---

## Typical workflows

### Interactive

```bash
terraform init
terraform plan
terraform apply
```

### CI/CD (best practice)

```bash
terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

---

## Summary table

| Aspect            | `plan`   | `apply`   |
| ----------------- | -------- | --------- |
| Reads config      | ✅        | ✅         |
| Talks to APIs     | ✅ (read) | ✅ (write) |
| Creates resources | ❌        | ✅         |
| Modifies state    | ❌        | ✅         |
| Requires approval | ❌        | ✅         |

---

## TL;DR

`terraform apply`:

* Executes real infrastructure changes
* Writes and locks state
* Is powerful and dangerous
* Should be gated, reviewed, and automated
