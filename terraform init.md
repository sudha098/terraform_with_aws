# What happens when you run `terraform init`

`terraform init` **prepares a working directory** for Terraform.
It does **not** create infrastructure and does **not** read or modify state (except backend setup).

Think of it as:

> “Download everything Terraform needs so this directory can work.”

---

## High-level steps `terraform init` performs

1. **Initializes the backend**
2. **Downloads provider plugins**
3. **Downloads modules**
4. **Creates the dependency lock file**
5. **Sets up the local Terraform working directory**

---

## Files and folders created (or updated)

### 1. `.terraform/` (directory)

This is the **local working directory cache**.
It is **machine-specific** and **should not be committed**.

Contents may include:

#### `.terraform/providers/`

* Downloaded **provider binaries**
* One per provider, per version, per platform

Example:

```
.terraform/providers/
└── registry.terraform.io/
    └── hashicorp/
        └── aws/
            └── 5.28.0/
```

Purpose:

* Avoids re-downloading providers every run
* Used during `plan` and `apply`

---

#### `.terraform/modules/`

* Cached copies of **modules** used by the configuration

Purpose:

* Resolves module sources (`git`, `registry`, local paths)
* Ensures Terraform runs against a known module snapshot

---

#### `.terraform/backend.tf` (internal)

* Internal metadata about the initialized backend
* **Not user-editable**
* Used to track backend configuration

---

### 2. `.terraform.lock.hcl` (file)

The **provider dependency lock file**.

Created or updated during `terraform init`.

Contains:

* Exact provider versions selected
* Provider source addresses
* Cryptographic checksums
* Platform-specific hashes

Purpose:

* Reproducible provider installs
* Supply-chain security
* Consistent behavior across machines and CI

✅ **Should be committed to Git**

---

### 3. Backend state connection (no local file in remote backends)

Depending on backend:

#### Local backend (default)

* No new files created yet
* State file appears later after `apply`

#### Remote backend (S3, GCS, Terraform Cloud, etc.)

* Validates credentials and connectivity
* May migrate state (if switching backends)
* No state file stored locally

---

### 4. Module downloads (side effect)

If your config contains:

```hcl
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
}
```

Terraform:

* Resolves the module
* Downloads it into `.terraform/modules/`
* Pins its version based on configuration

⚠️ Module versions are **not** stored in the lock file.

---

## Files NOT created by `terraform init`

Important to know what *doesn’t* happen:

❌ No infrastructure is created
❌ No resources are modified
❌ No `terraform.tfstate` (unless migrating backend)
❌ No secrets are generated
❌ No plans are executed

---

## Summary table

| File / Folder           | Created by `init` | Commit to Git | Purpose                             |
| ----------------------- | ----------------- | ------------- | ----------------------------------- |
| `.terraform/`           | ✅                 | ❌             | Local cache for providers & modules |
| `.terraform/providers/` | ✅                 | ❌             | Provider binaries                   |
| `.terraform/modules/`   | ✅                 | ❌             | Downloaded modules                  |
| `.terraform.lock.hcl`   | ✅                 | ✅             | Provider version & checksum locking |
| `terraform.tfstate`     | ❌                 | ❌             | Created after `apply`, not `init`   |

---

## TL;DR

`terraform init`:

* Sets up the **backend**
* Downloads **providers**
* Downloads **modules**
* Locks **provider versions**
* Prepares the directory to run `plan` and `apply`

