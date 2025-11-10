---
page_title: "Authentication with Databricks Unified Authentication"
---

# Authentication with Databricks Unified Authentication

-> **Note** This guide covers the recommended authentication method using Databricks unified authentication and CLI profiles. For other authentication methods, see the [main authentication documentation](../index.md#authentication).

Databricks unified authentication provides a consistent authentication experience across all Databricks tools, including the Terraform provider, Databricks CLI, SDKs, and IDEs. This approach simplifies credential management and follows security best practices.

## Overview

Unified authentication uses the Databricks CLI to manage credentials securely, storing them in `~/.databrickscfg`. The Terraform provider can then reference these credentials through named profiles, eliminating the need to manually generate access tokents or hardcode credentials in your Terraform configuration.

**Benefits:**
- **Centralized credential management**: Configure authentication once, use everywhere
- **Security**: Credentials stored in a single secure location, not in Terraform files
- **Flexibility**: Easy switching between different workspaces and accounts
- **OAuth support**: Native support for OAuth machine-to-machine (M2M) authentication
- **Cross-tool compatibility**: Same configuration works with CLI, Terraform, and SDKs

## Prerequisites

- Databricks CLI version 0.213.0 or later ([installation instructions](https://docs.databricks.com/en/dev-tools/cli/install.html))
- Terraform 1.1.5 or later
- Access to a Databricks workspace or account

## Setting Up Authentication

### Step 1: Install and Configure Databricks CLI

Install the Databricks CLI:

```bash
# macOS (Homebrew)
brew install databricks/tap/databricks

# Linux/macOS (curl)
curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh

# Windows (Winget)
winget install Databricks.DatabricksCLI
```

Verify installation:

```bash
databricks --version
```

### Step 2: Configure Authentication Profiles

#### For Workspace-Level Access

Configure a profile for your workspace using OAuth (recommended):

```bash
databricks auth login --host https://your-workspace.cloud.databricks.com
```

This will:
1. Open your browser for OAuth authentication
2. Create a profile in `~/.databrickscfg`
3. Store OAuth refresh tokens securely

Alternatively, configure manually with a Personal Access Token (PAT):

```bash
databricks configure --profile my-workspace
```

Enter your workspace URL and PAT when prompted.

#### For Account-Level Access

For account-level operations (creating workspaces, managing credentials, etc.), configure with OAuth service principal credentials:

```bash
databricks auth login \
  --host https://accounts.cloud.databricks.com \
  --account-id <your-account-id>
```

Or configure manually:

```bash
databricks configure --profile account --host https://accounts.cloud.databricks.com
```

### Step 3: Verify Configuration

Check your `~/.databrickscfg` file:

```bash
cat ~/.databrickscfg
```

Example configuration:

```ini
[DEFAULT]
host     = https://your-workspace.cloud.databricks.com
auth_type = databricks-cli

[account]
host       = https://accounts.cloud.databricks.com
account_id = 12345678-1234-1234-1234-123456789012
auth_type  = databricks-cli

[my-workspace]
host      = https://your-workspace.cloud.databricks.com
auth_type = databricks-cli
```

Test the connection:

```bash
databricks current-user me --profile DEFAULT
```

## Using with Terraform Provider

### Method 1: Profile-Based Authentication (Recommended)

Reference the profile name in your Terraform provider configuration:

```hcl
terraform {
  required_providers {
    databricks = {
      source = "databricks/databricks"
    }
  }
}

# Workspace-level provider
provider "databricks" {
  profile = "DEFAULT"
}

# Account-level provider for workspace provisioning
provider "databricks" {
  alias   = "mws"
  profile = "account"
}
```

### Method 2: Explicit auth_type Configuration

Use `auth_type = "databricks-cli"` to automatically use the DEFAULT profile:

```hcl
provider "databricks" {
  auth_type = "databricks-cli"
}
```

This is equivalent to:

```hcl
provider "databricks" {
  profile = "DEFAULT"
}
```

### Method 3: Environment Variables

Set the profile via environment variable:

```bash
export DATABRICKS_CONFIG_PROFILE="my-workspace"
terraform plan
```

Or specify auth type:

```bash
export DATABRICKS_AUTH_TYPE="databricks-cli"
terraform plan
```

## Complete Example: Provisioning a Workspace

This example demonstrates using unified authentication to provision an AWS Databricks workspace.

**Directory structure:**
```
terraform-project/
├── main.tf
├── providers.tf
├── variables.tf
└── terraform.tfvars
```

**providers.tf:**

```hcl
terraform {
  required_providers {
    databricks = {
      source = "databricks/databricks"
    }
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = var.region
}

# Account-level provider for creating workspace
provider "databricks" {
  alias   = "mws"
  profile = "account"
}
```

**variables.tf:**

```hcl
variable "databricks_account_id" {
  description = "Databricks account ID"
  type        = string
}

variable "region" {
  description = "AWS region"
  type        = string
  default     = "us-west-2"
}

variable "prefix" {
  description = "Prefix for resource names"
  type        = string
  default     = "my-workspace"
}
```

**main.tf:**

```hcl
# Cross-account IAM role for Databricks
data "databricks_aws_assume_role_policy" "this" {
  provider    = databricks.mws
  external_id = var.databricks_account_id
}

resource "aws_iam_role" "cross_account_role" {
  name               = "${var.prefix}-crossaccount"
  assume_role_policy = data.databricks_aws_assume_role_policy.this.json
}

data "databricks_aws_crossaccount_policy" "this" {
  provider = databricks.mws
}

resource "aws_iam_role_policy" "this" {
  name   = "${var.prefix}-policy"
  role   = aws_iam_role.cross_account_role.id
  policy = data.databricks_aws_crossaccount_policy.this.json
}

resource "databricks_mws_credentials" "this" {
  provider         = databricks.mws
  role_arn         = aws_iam_role.cross_account_role.arn
  credentials_name = "${var.prefix}-creds"
}

# Create workspace (simplified example)
resource "databricks_mws_workspaces" "this" {
  provider       = databricks.mws
  account_id     = var.databricks_account_id
  workspace_name = var.prefix
  aws_region     = var.region

  credentials_id = databricks_mws_credentials.this.credentials_id
  # storage_configuration_id and network_id would be defined separately
}

output "workspace_url" {
  value = databricks_mws_workspaces.this.workspace_url
}
```

Run Terraform:

```bash
terraform init
terraform plan
terraform apply
```

## Authentication for Service Principals (CI/CD)

For automated pipelines, use OAuth machine-to-machine (M2M) authentication with service principals.

### Step 1: Create Service Principal

```bash
databricks service-principals create --display-name "terraform-automation"
```

### Step 2: Generate OAuth Secret

```bash
databricks service-principals create-secret --id <service-principal-id>
```

Save the `client_id` and `client_secret` output.

### Step 3: Configure Profile with Service Principal

Create a profile in `~/.databrickscfg`:

```ini
[cicd]
host          = https://accounts.cloud.databricks.com
account_id    = 12345678-1234-1234-1234-123456789012
client_id     = <service-principal-application-id>
client_secret = <service-principal-secret>
auth_type     = oauth-m2m
```

Or use environment variables in CI/CD:

```bash
export DATABRICKS_HOST="https://accounts.cloud.databricks.com"
export DATABRICKS_ACCOUNT_ID="12345678-1234-1234-1234-123456789012"
export DATABRICKS_CLIENT_ID="<service-principal-application-id>"
export DATABRICKS_CLIENT_SECRET="<service-principal-secret>"
export DATABRICKS_AUTH_TYPE="oauth-m2m"
```

**Terraform configuration for CI/CD:**

```hcl
provider "databricks" {
  alias = "mws"
  # Authentication automatically picked up from environment variables
  # or uses the 'cicd' profile if DATABRICKS_CONFIG_PROFILE=cicd is set
}
```

## Authentication Priority and Resolution

The Terraform provider resolves authentication in the following order:

1. **Explicit provider configuration** (e.g., `host`, `token`, `client_id`)
2. **Environment variables** (e.g., `DATABRICKS_HOST`, `DATABRICKS_TOKEN`)
3. **Profile specified in provider** (`profile = "name"`)
4. **Profile from environment** (`DATABRICKS_CONFIG_PROFILE`)
5. **DEFAULT profile** in `~/.databrickscfg`
6. **Azure CLI authentication** (Azure only, when `auth_type = "azure-cli"`)
7. **Databricks CLI authentication** (when `auth_type = "databricks-cli"`)

## Best Practices

### Security

- **Use OAuth**: Prefer OAuth M2M for service principals over PAT tokens
- **Rotate credentials**: Regularly rotate service principal secrets
- **Limit scope**: Create workspace-specific service principals with minimal permissions
- **Protect .databrickscfg**: Ensure `~/.databrickscfg` has proper file permissions (600)
- **Never commit credentials**: Add `.databrickscfg` to `.gitignore`

### Profile Management

- **Use descriptive names**: Name profiles clearly (e.g., `prod-workspace`, `dev-account`)
- **Separate concerns**: Use different profiles for workspace-level and account-level operations
- **Document profiles**: Maintain a README documenting which profiles are used for what
- **Default profile**: Keep your most-used workspace as DEFAULT for convenience

### Terraform Configuration

- **Use profile references**: Always use `profile` in production Terraform configurations
- **Avoid inline credentials**: Never hardcode credentials in `.tf` files
- **Explicit auth_type**: Set `auth_type = "databricks-cli"` for clarity
- **Provider aliases**: Use provider aliases to manage multiple workspaces in one configuration

### Development vs Production

**Development (local):**
```hcl
provider "databricks" {
  profile = "dev-workspace"
}
```

**Production (CI/CD):**
```hcl
provider "databricks" {
  # Relies on environment variables set in CI/CD pipeline
  auth_type = "oauth-m2m"
}
```

## Troubleshooting

### Error: cannot configure default credentials

**Cause:** No valid authentication found.

**Solution:**
1. Verify `~/.databrickscfg` exists and contains valid profiles
2. Check the profile name matches: `databricks profiles list`
3. Ensure the profile has valid credentials: `databricks auth login`

### Error: databricks-cli auth: cannot get access token

**Cause:** OAuth tokens expired or invalid.

**Solution:**
```bash
# Re-authenticate
databricks auth login --profile <profile-name>
```

### Error: More than one authorization method configured

**Cause:** Multiple authentication methods specified (e.g., both `profile` and explicit `token`).

**Solution:** Remove conflicting authentication configurations. Use only one method:

```hcl
# Good - single method
provider "databricks" {
  profile = "my-workspace"
}

# Bad - conflicting methods
provider "databricks" {
  profile = "my-workspace"
  token   = "dapi..."  # Remove this!
}
```

### Verify Active Authentication

Check which credentials are being used:

```bash
# Check CLI configuration
databricks auth env

# Test connection
databricks current-user me --profile my-workspace

# Enable Terraform debug logging
export TF_LOG=DEBUG
terraform plan
```

## Migration from Legacy Authentication

If you're currently using explicit credentials in Terraform, migrate to unified authentication:

### Before (Legacy):

```hcl
provider "databricks" {
  host  = "https://workspace.cloud.databricks.com"
  token = var.databricks_token  # Don't do this!
}
```

### After (Unified Authentication):

1. Configure CLI profile:
```bash
databricks auth login --host https://workspace.cloud.databricks.com
```

2. Update Terraform:
```hcl
provider "databricks" {
  profile = "DEFAULT"
  # Or simply:
  # auth_type = "databricks-cli"
}
```

3. Remove credential variables from `variables.tf` and `terraform.tfvars`

## Related Resources

- [Databricks Unified Authentication Documentation](https://docs.databricks.com/en/dev-tools/auth/unified-auth.html)
- [Databricks CLI Documentation](https://docs.databricks.com/en/dev-tools/cli/index.html)
- [Databricks CLI Profiles](https://docs.databricks.com/en/dev-tools/cli/profiles.html)
- [Terraform Provider Authentication](../index.md#authentication)
- [Provisioning AWS Databricks workspace](aws-workspace.md)
- [Provisioning Azure Databricks workspace](azure-workspace.md)
- [OAuth Machine-to-Machine (M2M) Authentication](https://docs.databricks.com/en/dev-tools/auth/oauth-m2m.html)

