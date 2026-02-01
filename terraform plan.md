# `terraform plan` explained

`terraform plan` **calculates what Terraform *would* do** if you ran `terraform apply`.

It’s a **dry run**:

* No infrastructure is changed
* No state is modified
* No API calls that create/update/delete resources

Think of it as:

> “Show me the diff between reality and what my code says reality should be.”

---

## What happens during `terraform plan`

### 1. Loads configuration

Terraform reads:

* `*.tf` files
* Variables (`.tfvars`, env vars, defaults)
* Module code (from `.terraform/modules/`)

---

### 2. Loads current state

Terraform reads the **current state**:

* Local: `terraform.tfstate`
* Remote: fetched from backend (S3, GCS, Terraform Cloud, etc.)

State represents **what Terraform believes exists**.

---

### 3. Refreshes real infrastructure

Terraform queries provider APIs to check **actual resource state**:

* Reads AWS / GCP / Azure / etc.
* Compares live infrastructure vs state

This step can be skipped with:

```bash
terraform plan -refresh=false
```

---

### 4. Builds the dependency graph

Terraform:

* Resolves resource dependencies
* Determines creation/update/destruction order

---

### 5. Computes the execution plan

Terraform calculates:

* Resources to **create**
* Resources to **update**
* Resources to **destroy**
* Resources that are **unchanged**

---

### 6. Displays the plan

Output includes:

* `+` create
* `~` update
* `-` destroy
* `-/+` replace

Example:

```diff
+ aws_instance.web
~ aws_security_group.sg
- aws_s3_bucket.old
```

---

## Files created or modified by `terraform plan`

### By default: **none** ❌

`terraform plan`:

* Does **not** create infrastructure
* Does **not** write state
* Does **not** modify `.terraform.lock.hcl`

---

### Optional: saved plan file

If you run:

```bash
terraform plan -out=tfplan
```

Terraform creates:

| File     | Purpose               |
| -------- | --------------------- |
| `tfplan` | Binary execution plan |

Notes:

* This is a **binary file**
* Must be applied with the same config & state
* Often used in CI/CD pipelines

Example:

```bash
terraform apply tfplan
```

---

## What `terraform plan` depends on

| Input            | Required            |
| ---------------- | ------------------- |
| `terraform init` | ✅ must be run first |
| Provider plugins | ✅                   |
| Backend          | ✅                   |
| State            | ✅                   |
| Credentials      | ✅                   |

If init hasn’t been run:

```
Error: Backend initialization required
```

---

## Common flags

| Flag                    | What it does                      |
| ----------------------- | --------------------------------- |
| `-out=tfplan`           | Save plan to file                 |
| `-var-file=prod.tfvars` | Use specific vars                 |
| `-target=resource`      | Plan only specific resources      |
| `-refresh=false`        | Skip API refresh                  |
| `-destroy`              | Plan destruction of all resources |
| `-detailed-exitcode`    | CI-friendly exit codes            |

### `-detailed-exitcode`

* `0` → no changes
* `1` → error
* `2` → changes present

Perfect for pipelines 👌

---

## What plan does *not* do

❌ Apply changes
❌ Modify infrastructure
❌ Persist state changes
❌ Create resources
❌ Guarantee apply will succeed (APIs can still fail)

---

## Typical workflow

```bash
terraform init
terraform plan
terraform apply
```

In CI/CD:

```bash
terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

---

## TL;DR

`terraform plan`:

* Compares **desired state** vs **actual state**
* Shows exactly what will change
* Is safe, read-only, and essential
* Creates **no files unless you ask it to**

