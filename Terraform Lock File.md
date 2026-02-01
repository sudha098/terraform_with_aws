## Terraform Lock File (`.terraform.lock.hcl`)

This repository includes the Terraform lock file **`.terraform.lock.hcl`**, which is used to ensure consistent and secure Terraform executions across different environments and machines.

### What this file does

The lock file records the **exact provider versions** and **checksums** used by Terraform. This guarantees that everyone running Terraform—locally or in CI—uses the same provider binaries.

Specifically, it contains:

* Provider source addresses (e.g. `registry.terraform.io/hashicorp/aws`)
* Exact provider versions selected during `terraform init`
* Cryptographic checksums for provider binaries
* Platform-specific hashes for different OS/architectures

### Why this matters

* **Reproducibility** – Prevents unexpected changes caused by provider version drift
* **Security** – Verifies provider integrity and protects against tampering
* **Consistency** – Ensures identical behavior across developers and CI/CD pipelines

### What it does *not* contain

The lock file does **not** include:

* Terraform state
* Backend configuration
* Module versions
* Secrets or credentials

### When it is updated

The lock file is automatically managed by Terraform and may be updated when running:

* `terraform init`
* `terraform init -upgrade`

Manual edits are discouraged.

### Version control

✅ **This file should be committed to version control.**

Committing the lock file ensures stable and predictable Terraform runs and avoids “works on my machine” issues.

