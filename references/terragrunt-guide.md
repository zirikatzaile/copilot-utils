---
post_title: "Terragrunt for Azure: Project Structure, Units, Stacks, and OIDC Auth"
author1: "Platform Engineering Team"
post_slug: "terragrunt-azure-guide"
microsoft_alias: "platformeng"
featured_image: ""
categories: ["Infrastructure as Code", "Azure"]
tags: ["Terragrunt", "Terraform", "Azure", "OIDC", "IaC", "DevOps"]
ai_note: "This document was created with AI assistance."
summary: >
  A comprehensive guide to the Terragrunt project scaffolding in this repository:
  how the configuration hierarchy works, what units and stacks are, how remote
  modules and remote state are configured, and how OIDC authentication ties
  everything together for Azure deployments.
post_date: "2026-02-18"
---

## Overview

[Terragrunt](https://terragrunt.gruntwork.io/) is a thin orchestration wrapper
around Terraform / OpenTofu that solves three core problems at scale:

- **DRY configuration** – a single `root.hcl` file provides the backend,
  provider, and common tags for every unit in the repository.
- **Ordered, dependency-aware deployments** – `dependency {}` blocks and stacks
  let Terragrunt resolve execution order automatically.
- **Multi-environment promotion** – a layered directory structure separates what
  changes between environments (a few variables) from what stays the same (the
  Terraform module logic).

---

## Repository Layout

```text
Terragrunt/
├── root.hcl                        # Single source of truth: backend + provider + tags
├── terragrunt.hcl                  # Workspace-level entrypoint (thin wrapper)
├── .gitignore
│
├── _envcommon/                     # Shared Terragrunt configs (DRY defaults per component)
│   ├── networking.hcl
│   ├── aks.hcl
│   └── key-vault.hcl
│
├── catalog/
│   ├── tf-modules/                 # Stub Terraform modules (real ones live in a separate repo)
│   │   ├── networking/main.tf
│   │   ├── aks/main.tf
│   │   └── key-vault/main.tf
│   │
│   ├── units/                      # Reusable Terragrunt unit templates
│   │   ├── networking/terragrunt.hcl
│   │   ├── aks/terragrunt.hcl
│   │   └── key-vault/terragrunt.hcl
│   │
│   └── stacks/                     # Reusable stack definitions grouping multiple units
│       └── core-infrastructure/
│           └── terragrunt.stack.hcl
│
└── environments/
    ├── dev/
    │   ├── env.hcl                 # dev-specific: subscription ID, client ID, address space
    │   └── westeurope/
    │       ├── region.hcl          # region-specific: location, short suffix
    │       ├── terragrunt.stack.hcl  # stack orchestrator for this env/region
    │       ├── networking/terragrunt.hcl
    │       ├── aks/terragrunt.hcl
    │       └── key-vault/terragrunt.hcl
    ├── staging/  (same structure)
    └── prod/     (same structure)
```

---

## Key Concepts Illustrated

### Configuration Hierarchy and `find_in_parent_folders()`

Terragrunt walks up the directory tree to locate configuration files. Every
unit's `terragrunt.hcl` uses two `include` blocks:

```hcl
include "root" {
  path   = find_in_parent_folders("root.hcl")  # provider + backend
  expose = true
}

include "envcommon" {
  path           = find_in_parent_folders("_envcommon/networking.hcl")
  expose         = true
  merge_strategy = "deep"  # deep-merge inputs, not overwrite
}
```

This creates a three-layer hierarchy:

| Layer | File | Purpose |
|-------|------|---------|
| Root | `root.hcl` | Provider generation, remote state, common tags |
| Component common | `_envcommon/<component>.hcl` | Module source, default inputs |
| Environment live | `environments/<env>/<region>/<component>/terragrunt.hcl` | Environment overrides |

Values flow downward; lower layers can override upper ones.

---

### Units

A **unit** is one `terragrunt.hcl` file that wraps a single Terraform module.
It is the smallest deployable piece. In the catalog, a unit template looks like:

```hcl
terraform {
  source = "git::https://github.com/myorg/terraform-azure-modules.git//modules/networking?ref=v1.3.0"
}

inputs = {
  resource_group_name = "rg-networking-${local.environment}-${local.location_short}"
  vnet_name           = "vnet-${local.environment}-${local.location_short}-001"
}
```

Key points:

- The `//` separator in the Git URL tells Terragrunt where the Terraform module
  root starts inside the repository.
- The `?ref=v1.3.0` tag pins the module version, making deployments reproducible.
- Units in `environments/` override only the values that differ per environment;
  everything else is inherited from `_envcommon/`.

---

### Stacks

A **stack** (`terragrunt.stack.hcl`) groups multiple units into a deployable
set and wires values between them. Stacks can reference other stacks, enabling
layered reuse:

```hcl
# catalog/stacks/core-infrastructure/terragrunt.stack.hcl
unit "networking" {
  source = "git::https://github.com/myorg/infrastructure-catalog.git//catalog/units/networking?ref=v1.3.0"
  path   = "networking"
  values = {
    environment        = values.environment
    vnet_address_space = values.vnet_address_space
  }
}

unit "aks" {
  source = "git::https://github.com/myorg/infrastructure-catalog.git//catalog/units/aks?ref=v1.3.0"
  path   = "aks"
  values = {
    networking_path = "../networking"   # reference between sibling units
  }
}
```

The environment-level `terragrunt.stack.hcl` consumes this catalog stack:

```hcl
# environments/dev/westeurope/terragrunt.stack.hcl
stack "core_infra" {
  source = "git::https://github.com/myorg/infrastructure-catalog.git//catalog/stacks/core-infrastructure?ref=v1.3.0"
  path   = "."
  values = {
    environment        = "dev"
    vnet_address_space = "10.10.0.0/16"
    node_vm_size       = "Standard_D2s_v5"
  }
}
```

Running `terragrunt run --all apply` from `environments/dev/westeurope/`
deploys the entire stack in the correct dependency order.

---

### Remote Terraform Modules

All Terraform source modules live in a **separate repository** and are
referenced with a pinned Git tag. Terragrunt downloads them on `init`:

```hcl
terraform {
  # Long form – explicit registry URL:
  source = "tfr://registry.terraform.io/Azure/aks/azurerm?version=10.0.0"

  # Or Git shorthand pointing to a private module repository:
  source = "git::https://github.com/myorg/terraform-azure-modules.git//modules/networking?ref=v1.3.0"
}
```

Benefits:

- Module code and live configuration are in separate repositories with
  independent release cycles.
- Pinned tags (`?ref=v1.3.0`) ensure that a `plan` today gives the same result
  as a `plan` tomorrow.
- Upgrading a module is a one-line change in `_envcommon/<component>.hcl`,
  automatically promoted to all environments that include it.

---

### Remote State (Azure Storage Account)

The `remote_state` block in `root.hcl` configures Azure Storage as the
Terraform state backend. Terragrunt auto-generates a `backend.tf` file in
every unit's working directory:

```hcl
remote_state {
  backend = "azurerm"

  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }

  config = {
    resource_group_name  = "rg-tfstate-${local.environment}"
    storage_account_name = "sttfstate${local.environment}001"
    container_name       = "tfstate"

    # Unique key per unit: e.g. "environments/dev/westeurope/networking/terraform.tfstate"
    key = "${path_relative_to_include()}/terraform.tfstate"

    # Authenticates via Azure AD (OIDC) – no storage account key needed
    use_azuread_auth = true
  }
}
```

Important aspects:

- `path_relative_to_include()` derives a unique state key from the directory
  path, preventing state collisions between units.
- `use_azuread_auth = true` means access to the storage account is controlled
  by RBAC (`Storage Blob Data Contributor` on the container) rather than a
  shared access key.
- The storage account and container must be **pre-provisioned** (bootstrap)
  before running any Terragrunt commands. This is a common bootstrap step
  done once per environment.

---

### OIDC Authentication for Azure

OIDC (federated credentials) removes long-lived secrets from CI/CD pipelines.
The flow is:

```text
CI/CD Runner  ──(OIDC token)──►  Azure AD  ──(short-lived token)──►  Azure ARM API
```

**What is configured in code (`root.hcl`):**

```hcl
generate "provider_azure" {
  contents = <<-EOF
    provider "azurerm" {
      subscription_id = "${local.subscription_id}"
      tenant_id       = "${local.tenant_id}"
      client_id       = "${local.client_id}"
      use_oidc        = true   # reads ARM_OIDC_TOKEN at runtime
      features {}
    }
  EOF
}
```

**What is set by the CI/CD pipeline (environment variables – never stored in code):**

| Variable | Source | Description |
|----------|--------|-------------|
| `ARM_CLIENT_ID` | Pipeline variable / secret | Service Principal application (client) ID |
| `ARM_TENANT_ID` | Pipeline variable | Azure AD tenant ID |
| `ARM_SUBSCRIPTION_ID` | Pipeline variable | Target Azure subscription ID |
| `ARM_USE_OIDC` | Pipeline variable (`true`) | Signals the provider to use OIDC |
| `ARM_OIDC_TOKEN` | Injected by CI runtime | The OIDC JWT token from the identity provider |

**Azure prerequisites:**

- A Service Principal (or User-Assigned Managed Identity) with a
  **Federated Identity Credential** pointing to the CI/CD provider
  (e.g. `repo:myorg/infra:environment:dev` for GitHub Actions, or a
  service connection for Azure DevOps).
- The SP must have `Contributor` (or scoped roles) on the target subscription
  and `Storage Blob Data Contributor` on the Terraform state container.

**Local development** uses `az login` + `ARM_USE_OIDC=false` (or defaults),
so engineers never handle raw credentials on their workstations.

---

### Inter-Unit Dependencies

A unit can read the Terraform outputs of another unit via `dependency {}`:

```hcl
dependency "networking" {
  config_path = "../networking"   # path to the other unit's terragrunt.hcl

  mock_outputs = {
    subnet_ids = { app = "/subscriptions/.../subnets/app" }
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

inputs = {
  vnet_subnet_id = dependency.networking.outputs.subnet_ids["app"]
}
```

- **Real outputs** are read from the remote state file of the dependency.
- **Mock outputs** allow `plan` to succeed before the dependency has been applied,
  which is essential in CI pipelines that run plan in isolation.
- Terragrunt enforces apply order automatically when using `run --all`.

---

## Common Terragrunt Commands

### Unit Commands

Commands run against a single `terragrunt.hcl` unit from that unit's directory:

| Command | Description |
| ------- | ----------- |
| `terragrunt plan` | Plan a single unit |
| `terragrunt apply` | Apply a single unit |
| `terragrunt destroy` | Destroy resources managed by a single unit |
| `terragrunt output` | Print outputs of a single unit |
| `terragrunt hcl validate --inputs` | Check that all inputs match the module's declared variables |

### Multi-Unit Commands (`run --all`)

These commands discover and operate on every unit found under the current
working directory. Terragrunt respects `dependency {}` ordering automatically.
Running from `environments/dev/westeurope/` scopes execution to dev West
Europe units only.

| Command | Scope | Description |
| ------- | ----- | ----------- |
| `terragrunt run --all plan` | All units below CWD | Plan every unit in dependency order |
| `terragrunt run --all apply` | All units below CWD | Apply all units in dependency order |
| `terragrunt run --all destroy` | All units below CWD | Destroy all units in reverse dependency order |
| `terragrunt run --all output` | All units below CWD | Print outputs from every unit |
| `terragrunt run --all apply --queue-include-dir environments/dev` | Filtered set | Apply only units under `environments/dev` |
| `terragrunt run --all plan --non-interactive` | All units below CWD | Non-interactive mode for CI/CD pipelines |

### Explicit Stack Commands (`stack *`)

These commands operate on `terragrunt.stack.hcl` files and their generated
units. Run them from the directory containing the `terragrunt.stack.hcl` file
(e.g. `environments/dev/westeurope/`).

| Command | Description |
| ------- | ----------- |
| `terragrunt stack generate` | Parse `terragrunt.stack.hcl` and generate the `.terragrunt-stack/` directory with a `terragrunt.hcl` for each declared unit |
| `terragrunt stack run plan` | Generate the stack (if needed) then run `plan` across every unit in the stack, respecting dependency order |
| `terragrunt stack run apply` | Generate the stack (if needed) then run `apply` across every unit in the stack, respecting dependency order |
| `terragrunt stack run destroy` | Run `destroy` across every unit in the stack in reverse dependency order |
| `terragrunt stack output` | Print a single aggregated map of all outputs from all units in the stack |
| `terragrunt stack generate --parallelism 4` | Generate the stack using up to 4 parallel workers |
| `terragrunt stack generate --no-stack-validate` | Generate without validating the stack schema (useful for debugging) |

The `stack run` commands are the recommended way to operate on an explicit
`terragrunt.stack.hcl` definition, whereas `run --all` is suited for
discovering and running against ad-hoc collections of units without a
stack file.

---

## Environment Promotion Pattern

To promote a configuration change from `dev` to `staging` to `prod`:

1. Change the value in the relevant `_envcommon/<component>.hcl` or
   `catalog/units/<unit>/terragrunt.hcl` (applies to all environments).
2. Or change a value in `environments/<env>/env.hcl` for an environment-specific
   override (applies only to that environment).
3. Run `terragrunt run --all plan` in each environment directory in order.
4. Review and apply each environment sequentially in the CI/CD pipeline.

Because `env.hcl` and `region.hcl` are the only files that differ between
environments, the configuration delta is always small and reviewable.

---

## Local Development

### Authentication

When working locally, skip OIDC and authenticate with the Azure CLI instead.
The `azurerm` provider and the `azurerm` backend both honour `az login`
credentials automatically:

```bash
# Interactive browser login
az login

# Switch to the target subscription
az account set --subscription "<subscription-id>"

# Verify the active account
az account show
```

Set the following shell variables so Terragrunt / Terraform can resolve the
non-OIDC provider arguments that are otherwise supplied by the pipeline:

```bash
export ARM_SUBSCRIPTION_ID="<subscription-id>"
export ARM_TENANT_ID="<tenant-id>"
export ARM_CLIENT_ID=""      # leave empty – az login identity is used
export ARM_USE_OIDC="false"  # disable OIDC; use az login token instead
```

---

### Overriding the Module Source for Local Testing

When iterating on a Terraform module locally, avoid pushing a Git tag for every
change. Use the `--source` flag to point Terragrunt at a local checkout:

```bash
# Run against a single unit, using a local module path
cd environments/dev/westeurope/networking
terragrunt plan --source ../../../../catalog/tf-modules/networking

# Run against all units, replacing the Git source with a local modules root
cd environments/dev/westeurope
terragrunt run --all plan --source /path/to/local/terraform-azure-modules
```

Terragrunt replaces only the part after `//` in the original `source` URL, so
the correct sub-directory is still appended automatically.

---

### Inspecting the Resolved Configuration

Use `render` to see the fully merged Terragrunt configuration for a unit,
including all `include` and `locals` expansions. This is invaluable for
debugging why an input has an unexpected value:

```bash
# Print the rendered config for the current unit as JSON to stdout
terragrunt render --json

# Write a terragrunt.rendered.json file next to every terragrunt.hcl found
# below the current directory
terragrunt render --all --json -w
```

The `--inputs-debug` flag additionally dumps the final input values that
Terragrunt will pass to Terraform:

```bash
terragrunt plan --inputs-debug
```

---

### Visualising the Dependency Graph

Before applying a large change, inspect which units will be affected and in
what order:

```bash
# List all units and their dependencies as a tree (requires CWD to be a
# directory containing multiple units, e.g. environments/dev/westeurope/)
terragrunt list --dag --tree

# Output the full DAG as JSON (useful for scripting)
terragrunt find --dag --json --dependencies

# Render the graph as an SVG (requires Graphviz installed locally)
terragrunt dag graph | dot -Tsvg > graph.svg
open graph.svg
```

---

### Debugging and Verbose Logging

```bash
# Increase log verbosity to see every Terraform call and input resolution
terragrunt plan --log-level debug

# Combine debug logging with inputs dump
terragrunt apply --log-level debug --inputs-debug

# Alternatively, set environment variables for the whole shell session
export TG_LOG="debug"
export TG_DEBUG_INPUTS="true"
```

---

### Validating Inputs Without Applying

Catch mismatches between Terragrunt `inputs` and Terraform `variable` blocks
before touching any real infrastructure:

```bash
# Validate a single unit
terragrunt hcl validate --inputs

# Validate every unit below the current directory
terragrunt run --all validate-inputs
```

---

### Clearing the Cache

Terragrunt caches downloaded modules under `.terragrunt-cache/`. If you see
stale module code or provider plugin issues, wipe the cache:

```bash
# Remove all cache directories under the current path
find . -type d -name ".terragrunt-cache" -prune -exec rm -rf {} \;

# Force a fresh init on the next run (auto-init is on by default)
terragrunt init --reconfigure
```

---

### Recommended Local Workflow

A typical iteration loop for changing a single unit:

```bash
# 1. Authenticate
az login && az account set --subscription "<id>"

# 2. Navigate to the unit
cd environments/dev/westeurope/networking

# 3. Validate inputs and syntax
terragrunt run validate --source <source-to-tf-modules>

# 4. Plan (mock outputs let this work even if dependencies aren't applied)
terragrunt run plan --source <source-to-tf-modules>
```

For a full environment, replace step 2–6 with:

```bash
cd environments/dev/westeurope

# Plan and inspect the dependency order first
terragrunt list --dag --tree

# Stack-aware: generate then apply in dependency order
terragrunt stack run apply

# Or, for ad-hoc unit discovery without a stack file:
terragrunt run --all apply
```

---

## Bootstrap Checklist

Before the first `terragrunt apply`, complete the following once per environment:

- Create the Azure Storage Account and blob container for Terraform state.
- Assign `Storage Blob Data Contributor` to the CI Service Principal on the container.
- Register the federated credential on the Service Principal for the CI/CD provider.
- Populate pipeline variables: `ARM_CLIENT_ID`, `ARM_TENANT_ID`,
  `ARM_SUBSCRIPTION_ID`, `ARM_USE_OIDC=true`.
- Update `env.hcl` with the correct `subscription_id`, `tenant_id`, and
  `client_id` (or rely on `get_env()` to read them from the pipeline).
