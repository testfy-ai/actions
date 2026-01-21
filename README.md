# Testfy.ai GitHub Actions

A collection of reusable GitHub Actions for Terraform and Azure workflows.

## Actions

| Action | Description |
|--------|-------------|
| [setup-azure-terraform](#setup-azure-terraform) | Setup Azure CLI, Terraform, and login with OIDC |
| [terraform-run](#terraform-run) | Run terraform fmt, init, validate, plan, and apply |
| [terraform-summary](#terraform-summary) | Create a validation summary for PRs |
| [terraform-plan-extract](#terraform-plan-extract) | Extract and parse plan changes from JSON |

---

## setup-azure-terraform

Setup Azure CLI, Terraform, and login to Azure with OIDC authentication.

### Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `terraform-version` | Terraform version to install | No | `latest` |
| `terraform-wrapper` | Enable terraform wrapper for output parsing | No | `true` |
| `azure-cli-version` | Azure CLI version to install | No | `latest` |
| `azure-client-id` | Azure Client ID for OIDC | **Yes** | - |
| `azure-tenant-id` | Azure Tenant ID | **Yes** | - |
| `azure-subscription-id` | Azure Subscription ID | **Yes** | - |

### Outputs

| Output | Description |
|--------|-------------|
| `terraform-version` | The installed Terraform version |
| `azure-cli-version` | The installed Azure CLI version |

### Example

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4

      - name: Setup Azure and Terraform
        uses: testfy-ai/actions/setup-azure-terraform@main
        with:
          azure-client-id: ${{ secrets.AZURE_CLIENT_ID }}
          azure-tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          azure-subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          terraform-version: '1.7.0'
```

---

## terraform-run

Run terraform format check, init, validate, and optionally plan/apply.

### Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| **Working directory** |
| `working-directory` | Working directory for Terraform | No | `.` |
| **Format check options** |
| `fail-on-fmt-error` | Fail the workflow if terraform fmt check fails | No | `false` |
| **Init options** |
| `backend-config` | Backend configuration options for terraform init | No | `''` |
| `init-upgrade` | Run terraform init with -upgrade flag | No | `false` |
| `init-reconfigure` | Run terraform init with -reconfigure flag | No | `false` |
| **Plan options** |
| `run-plan` | Run terraform plan after init | No | `true` |
| `plan-file` | Output plan file name | No | `tfplan` |
| `plan-destroy` | Run destroy plan instead of apply plan | No | `false` |
| `plan-generate-json` | Generate JSON output of the plan | No | `true` |
| **Apply options** |
| `run-apply` | Run terraform apply | No | `false` |
| `use-plan-file` | Path to existing plan file to apply (skips plan step) | No | `''` |
| `apply-output-file` | Output file name for apply output | No | `apply-output.txt` |

### Outputs

| Output | Description |
|--------|-------------|
| `fmt-outcome` | Format check outcome (`success`, `failure`) |
| `init-outcome` | Init outcome (`success`, `failure`) |
| `validate-outcome` | Validate outcome (`success`, `failure`) |
| `plan-outcome` | Plan outcome (`success`, `failure`, `skipped`) |
| `apply-outcome` | Apply outcome (`success`, `failure`, `skipped`) |
| `plan-exitcode` | Terraform plan exit code (`0`=no changes, `1`=error, `2`=changes) |
| `plan-has-changes` | Whether the plan has changes (`true`/`false`) |
| `plan-output-file` | Path to the plan output text file |
| `plan-file` | Path to the plan binary file |
| `plan-json-file` | Path to the plan JSON file (if generated) |
| `apply-output-file` | Path to the apply output file |

### Example: Basic Plan

```yaml
jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Azure and Terraform
        uses: testfy-ai/actions/setup-azure-terraform@main
        with:
          azure-client-id: ${{ secrets.AZURE_CLIENT_ID }}
          azure-tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          azure-subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Terraform Plan
        id: terraform
        uses: testfy-ai/actions/terraform-run@main
        with:
          working-directory: ./infra
          backend-config: '-backend-config=key=${{ github.ref_name }}.tfstate'
```

### Example: Matrix Strategy (Multiple Environments)

```yaml
jobs:
  plan:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [dev, staging, prod]
    steps:
      - uses: actions/checkout@v4

      - name: Setup Azure and Terraform
        uses: testfy-ai/actions/setup-azure-terraform@main
        with:
          azure-client-id: ${{ secrets.AZURE_CLIENT_ID }}
          azure-tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          azure-subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Terraform Plan
        id: terraform
        uses: testfy-ai/actions/terraform-run@main
        with:
          working-directory: ./envs/${{ matrix.environment }}
          fail-on-fmt-error: true

      - name: Create Summary
        uses: testfy-ai/actions/terraform-summary@main
        with:
          environment: ${{ matrix.environment }}
          working-directory: ./envs/${{ matrix.environment }}
          fmt-outcome: ${{ steps.terraform.outputs.fmt-outcome }}
          init-outcome: ${{ steps.terraform.outputs.init-outcome }}
          validate-outcome: ${{ steps.terraform.outputs.validate-outcome }}
          plan-outcome: ${{ steps.terraform.outputs.plan-outcome }}
          plan-has-changes: ${{ steps.terraform.outputs.plan-has-changes }}
```

### Example: Plan and Apply

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Setup Azure and Terraform
        uses: testfy-ai/actions/setup-azure-terraform@main
        with:
          azure-client-id: ${{ secrets.AZURE_CLIENT_ID }}
          azure-tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          azure-subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Terraform Apply
        uses: testfy-ai/actions/terraform-run@main
        with:
          working-directory: ./infra
          run-apply: true
```

---

## terraform-summary

Create a validation summary of Terraform steps for PRs. Writes to GitHub Job Summary and outputs a markdown file.

### Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `environment` | Environment name | **Yes** | - |
| `working-directory` | Working directory where plan-output.txt is located | **Yes** | - |
| `fmt-outcome` | Format check outcome (`success`, `failure`) | **Yes** | - |
| `init-outcome` | Init outcome (`success`, `failure`) | **Yes** | - |
| `validate-outcome` | Validate outcome (`success`, `failure`) | **Yes** | - |
| `plan-outcome` | Plan outcome (`success`, `failure`, `skipped`) | **Yes** | - |
| `plan-has-changes` | Whether the plan has changes (`true`/`false`) | No | `false` |

### Outputs

| Output | Description |
|--------|-------------|
| `result` | Overall result (`success`, `failure`) |
| `summary-file` | Path to the summary markdown file |

### Example

```yaml
- name: Create Summary
  uses: testfy-ai/actions/terraform-summary@main
  with:
    environment: production
    working-directory: ./infra
    fmt-outcome: ${{ steps.terraform.outputs.fmt-outcome }}
    init-outcome: ${{ steps.terraform.outputs.init-outcome }}
    validate-outcome: ${{ steps.terraform.outputs.validate-outcome }}
    plan-outcome: ${{ steps.terraform.outputs.plan-outcome }}
    plan-has-changes: ${{ steps.terraform.outputs.plan-has-changes }}
```

### Summary Output

The action generates a summary table like this:

| Step | Status |
|------|--------|
| Format | :white_check_mark: |
| Init | :white_check_mark: |
| Validate | :white_check_mark: |
| Plan | :warning: Changes detected |

---

## terraform-plan-extract

Extract and parse Terraform plan changes from JSON output. Useful for PR comments and notifications.

### Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `environment` | Environment name | **Yes** | - |
| `working-directory` | Working directory where plan JSON is located | **Yes** | - |
| `plan-file` | Plan JSON file name | No | `tfplan.json` |
| `plan-type` | Type of plan (`apply` or `destroy`) | No | `apply` |

### Outputs

| Output | Description |
|--------|-------------|
| **Summary** |
| `summary` | Human-readable summary (e.g., `+2 to add, ~1 to change`) |
| `has-changes` | Whether the plan has any changes (`true`/`false`) |
| `has-resources` | Whether there are any resources affected (`true`/`false`) |
| **Resource counts** |
| `add` | Number of resources to add |
| `change` | Number of resources to change |
| `destroy` | Number of resources to destroy |
| **Resource lists** |
| `resources-to-add` | List of resources to add (newline-separated) |
| `resources-to-change` | List of resources to change (newline-separated) |
| `resources-to-destroy` | List of resources to destroy (newline-separated) |

### Example

```yaml
- name: Terraform Plan
  id: terraform
  uses: testfy-ai/actions/terraform-run@main
  with:
    working-directory: ./infra

- name: Extract Plan Details
  id: extract
  uses: testfy-ai/actions/terraform-plan-extract@main
  with:
    environment: production
    working-directory: ./infra
    plan-file: tfplan.json

- name: Comment on PR
  if: github.event_name == 'pull_request' && steps.extract.outputs.has-changes == 'true'
  uses: actions/github-script@v8
  with:
    script: |
      const summary = `${{ steps.extract.outputs.summary }}`;
      const add = `${{ steps.extract.outputs.resources-to-add }}`;
      const change = `${{ steps.extract.outputs.resources-to-change }}`;
      const destroy = `${{ steps.extract.outputs.resources-to-destroy }}`;

      let body = `## Terraform Plan: ${summary}\n\n`;

      if (add) body += `### Resources to Add\n\`\`\`\n${add}\n\`\`\`\n\n`;
      if (change) body += `### Resources to Change\n\`\`\`\n${change}\n\`\`\`\n\n`;
      if (destroy) body += `### Resources to Destroy\n\`\`\`\n${destroy}\n\`\`\`\n\n`;

      github.rest.issues.createComment({
        owner: context.repo.owner,
        repo: context.repo.repo,
        issue_number: context.issue.number,
        body: body
      });
```

### Output Examples

**Scenario: Adding 3 resources, updating 1, deleting 2**
```
summary: "+3 to add, ~1 to change, -2 to destroy"
has-changes: "true"
add: "3"
change: "1"
destroy: "2"
resources-to-add: |
  azurerm_resource_group.main
  azurerm_storage_account.data
  azurerm_key_vault.secrets
resources-to-change: |
  azurerm_virtual_network.vnet
resources-to-destroy: |
  azurerm_storage_container.old
  azurerm_cosmosdb_account.legacy
```

**Scenario: No changes**
```
summary: "No changes"
has-changes: "false"
add: "0"
change: "0"
destroy: "0"
```

---

## Complete Workflow Examples

### PR Validation (Matrix Strategy)

Validate all environments on pull requests:

```yaml
name: Terraform PR Validation

on:
  pull_request:
    branches: [main, develop]

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  validate:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        environment: [shared, dev, test, stage, prod]
    steps:
      - uses: actions/checkout@v4

      - name: Setup Azure and Terraform
        uses: testfy-ai/actions/setup-azure-terraform@main
        with:
          azure-client-id: ${{ secrets.AZURE_CLIENT_ID }}
          azure-tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          azure-subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Terraform Plan
        id: terraform
        uses: testfy-ai/actions/terraform-run@main
        with:
          working-directory: ./envs/${{ matrix.environment }}
          fail-on-fmt-error: true

      - name: Extract Plan
        id: extract
        if: steps.terraform.outputs.plan-has-changes == 'true'
        uses: testfy-ai/actions/terraform-plan-extract@main
        with:
          environment: ${{ matrix.environment }}
          working-directory: ./envs/${{ matrix.environment }}

      - name: Create Summary
        uses: testfy-ai/actions/terraform-summary@main
        with:
          environment: ${{ matrix.environment }}
          working-directory: ./envs/${{ matrix.environment }}
          fmt-outcome: ${{ steps.terraform.outputs.fmt-outcome }}
          init-outcome: ${{ steps.terraform.outputs.init-outcome }}
          validate-outcome: ${{ steps.terraform.outputs.validate-outcome }}
          plan-outcome: ${{ steps.terraform.outputs.plan-outcome }}
          plan-has-changes: ${{ steps.terraform.outputs.plan-has-changes }}
```

### Manual Deployment (workflow_dispatch)

Deploy infrastructure manually with environment and action selection. Uses a two-job approach: plan job runs first (for plan/destroy), then apply job uses the saved plan file.

```yaml
name: Terraform Deploy

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to manage'
        required: true
        type: choice
        options:
          - shared
          - sand
          - dev
          - test
          - stage
          - prod
      action:
        description: 'Action to perform'
        required: true
        type: choice
        options:
          - plan
          - apply
          - destroy

permissions:
  id-token: write
  contents: read

jobs:
  plan:
    if: inputs.action == 'plan' || inputs.action == 'destroy'
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - uses: actions/checkout@v4

      - name: Setup Azure and Terraform
        uses: testfy-ai/actions/setup-azure-terraform@main
        with:
          azure-client-id: ${{ secrets.AZURE_CLIENT_ID }}
          azure-tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          azure-subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Terraform Plan
        id: terraform
        uses: testfy-ai/actions/terraform-run@main
        with:
          working-directory: ./envs/${{ inputs.environment }}
          plan-destroy: ${{ inputs.action == 'destroy' }}

      - name: Extract Plan
        id: extract
        if: steps.terraform.outputs.plan-has-changes == 'true'
        uses: testfy-ai/actions/terraform-plan-extract@main
        with:
          environment: ${{ inputs.environment }}
          working-directory: ./envs/${{ inputs.environment }}

      - name: Create Summary
        uses: testfy-ai/actions/terraform-summary@main
        with:
          environment: ${{ inputs.environment }}
          working-directory: ./envs/${{ inputs.environment }}
          fmt-outcome: ${{ steps.terraform.outputs.fmt-outcome }}
          init-outcome: ${{ steps.terraform.outputs.init-outcome }}
          validate-outcome: ${{ steps.terraform.outputs.validate-outcome }}
          plan-outcome: ${{ steps.terraform.outputs.plan-outcome }}
          plan-has-changes: ${{ steps.terraform.outputs.plan-has-changes }}

      - name: Upload Plan
        uses: actions/upload-artifact@v4
        with:
          name: tfplan-${{ inputs.environment }}
          path: ./envs/${{ inputs.environment }}/tfplan

  apply:
    if: inputs.action == 'apply'
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - uses: actions/checkout@v4

      - name: Setup Azure and Terraform
        uses: testfy-ai/actions/setup-azure-terraform@main
        with:
          azure-client-id: ${{ secrets.AZURE_CLIENT_ID }}
          azure-tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          azure-subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Download Plan
        uses: actions/download-artifact@v4
        with:
          name: tfplan-${{ inputs.environment }}
          path: ./envs/${{ inputs.environment }}

      - name: Terraform Apply
        uses: testfy-ai/actions/terraform-run@main
        with:
          working-directory: ./envs/${{ inputs.environment }}
          run-apply: true
          use-plan-file: tfplan
```

### CI/CD Pipeline (PR + Auto Deploy)

Full CI/CD with PR validation and automatic deployment on merge:

```yaml
name: Terraform CI/CD

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  plan:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        environment: [dev, stage, prod]
    steps:
      - uses: actions/checkout@v4

      - name: Setup Azure and Terraform
        uses: testfy-ai/actions/setup-azure-terraform@main
        with:
          azure-client-id: ${{ secrets.AZURE_CLIENT_ID }}
          azure-tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          azure-subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Terraform Plan
        id: terraform
        uses: testfy-ai/actions/terraform-run@main
        with:
          working-directory: ./envs/${{ matrix.environment }}
          fail-on-fmt-error: true

      - name: Create Summary
        uses: testfy-ai/actions/terraform-summary@main
        with:
          environment: ${{ matrix.environment }}
          working-directory: ./envs/${{ matrix.environment }}
          fmt-outcome: ${{ steps.terraform.outputs.fmt-outcome }}
          init-outcome: ${{ steps.terraform.outputs.init-outcome }}
          validate-outcome: ${{ steps.terraform.outputs.validate-outcome }}
          plan-outcome: ${{ steps.terraform.outputs.plan-outcome }}
          plan-has-changes: ${{ steps.terraform.outputs.plan-has-changes }}

      - name: Upload Plan
        uses: actions/upload-artifact@v4
        with:
          name: plan-${{ matrix.environment }}
          path: |
            ./envs/${{ matrix.environment }}/tfplan
            ./envs/${{ matrix.environment }}/tfplan.json

  deploy:
    needs: plan
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Setup Azure and Terraform
        uses: testfy-ai/actions/setup-azure-terraform@main
        with:
          azure-client-id: ${{ secrets.AZURE_CLIENT_ID }}
          azure-tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          azure-subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Download Plan
        uses: actions/download-artifact@v4
        with:
          name: plan-prod
          path: ./envs/prod

      - name: Terraform Apply
        uses: testfy-ai/actions/terraform-run@main
        with:
          working-directory: ./envs/prod
          run-apply: true
          use-plan-file: tfplan
```

## License

MIT
