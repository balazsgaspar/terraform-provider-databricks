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

- Databricks CLI version 0.205 or later ([installation instructions](https://docs.databricks.com/en/dev-tools/cli/install.html))
- Terraform 1.1.5 or later
- Access to a Databricks workspace or account

## Setting Up Authentication
The steps below are an example for 

### Step 1: Install and Configure Databricks CLI

Install the Databricks CLI:

```bash
# macOS (Homebrew)
brew tap databricks/tap
brew install databricks

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

Configure a profile for your workspace using OAuth (recommended). For the host parameter, use the base URL of the workspace from your browser.

```bash
databricks auth login --host https://your-workspace.cloud.databricks.com
```
When prompted, provide a name for the authentication profile. Replace my-workspace in the examples below with your actual workspace profile name.

This will:
1. Open your browser for OAuth authentication
2. Create a profile in `~/.databrickscfg`
3. Store OAuth refresh tokens securely

Alternatively, you can manually configure a Personal Access Token ([legacy](https://docs.databricks.com/aws/en/dev-tools/auth/pat)):

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

When prompted, provide a name for the authentication profile. Replace my-account in the examples below with your actual account profile name.


Or configure a PAT token ([legacy](https://docs.databricks.com/aws/en/dev-tools/auth/pat)):

```bash
databricks configure --profile my-account --host https://accounts.cloud.databricks.com
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

[my-account]
host       = https://accounts.cloud.databricks.com
account_id = 12345678-1234-1234-1234-123456789012
auth_type  = databricks-cli

[my-workspace]
host      = https://your-workspace.cloud.databricks.com
auth_type = databricks-cli
```

Test the connection to the workspace endpoint:

```bash
databricks current-user me --profile my-workspace
```

Test the connection to the account endpoint (replace <user-id> with the ID returned by the previous command):

```bash
databricks account users-v2 get <user-id>
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
  profile = "my-workspace"
}

# Account-level provider
provider "databricks" {
  alias   = "mws"
  profile = "my-account"
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
```

Or specify auth type:

```bash
export DATABRICKS_AUTH_TYPE="databricks-cli"
```

## Authentication for Service Principals (CI/CD)

For automated pipelines, use [OAuth machine-to-machine (M2M) authentication](https://docs.databricks.com/aws/en/dev-tools/auth/oauth-m2m) with service principals.
Use this method for deployments managed via automated CI/CD pipelines. Make sure that 

### Step 1: Create Service Principal

```bash
databricks service-principals create --display-name "terraform-automation"
```

### Step 2: Generate OAuth Secret

Replace <service-principal-id> with the ID returned by the previous create command.

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

## Related Resources

- [Terraform Provider Authentication](../index.md#authentication)
- [Provisioning AWS Databricks workspace](aws-workspace.md)
- [Provisioning Azure Databricks workspace](azure-workspace.md)
- [Databricks Unified Authentication Documentation](https://docs.databricks.com/en/dev-tools/auth/unified-auth.html)
- [Databricks CLI Documentation](https://docs.databricks.com/en/dev-tools/cli/index.html)
- [Databricks CLI Profiles](https://docs.databricks.com/en/dev-tools/cli/profiles.html)
- [OAuth Machine-to-Machine (M2M) Authentication](https://docs.databricks.com/en/dev-tools/auth/oauth-m2m.html)

