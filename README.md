# GitHub Action Templates

Reusable GitHub Actions workflows and composite actions for deploying Azure Landing Zone (ALZ)
infrastructure and applications. This is a GitHub Actions translation of the
[Remo-L5/azure-devops-pipeline-templates](https://github.com/Remo-L5/azure-devops-pipeline-templates)
Azure DevOps pipeline templates.

It supports multiple project types — **Terraform**, **API Operations (APIOps toolkit and CLI)**,
**Azure API Center**, and **.NET** — orchestrated through composable, reusable workflows.

## Repository structure

```
.github/
├── workflows/                         # Reusable workflows (workflow_call)
│   ├── terraform.yml                  # Terraform plan + apply
│   ├── ci.yml                         # Terraform fmt/validate + review plan (PRs)
│   ├── dotnet.yml                     # Build + publish .NET apps
│   ├── api-ops.yml                    # APIOps toolkit plan/deploy
│   ├── api-ops-cli.yml                # APIOps CLI plan/deploy
│   └── api-center.yml                 # API Center validate/register
└── actions/                           # Composite actions (step-level helpers)
    ├── terraform-init/
    ├── terraform-plan/
    ├── terraform-apply/
    ├── terraform-workspace/
    ├── az-storage-table-get-record/   # Subscription vending support
    ├── apiops-publisher/              # APIOps toolkit (plan via dry-run, or deploy)
    ├── apiops-cli/                    # APIOps CLI (plan via dry-run, or deploy)
    ├── api-center-validate/
    └── api-center-register/
examples/
├── example-cd.yml                      # Example CD caller wiring multiple projects
└── example-ci.yml                      # Example CI caller
```

## How the Azure DevOps concepts map to GitHub Actions

| Azure DevOps | GitHub Actions |
|---|---|
| Stage template (`stages:`) | Reusable workflow (`on: workflow_call`) |
| Step template (`steps:`) | Composite action (`runs: using: composite`) |
| `cd-template.yaml` `projects[]` + `dependencies[]` | Caller workflow with jobs + `needs:` (see `example-cd.yml`) |
| Service connection + `AzureCLI@2` | `azure/login@v2` with OIDC (federated credentials) |
| Variable group | Repository/Environment **secrets** and **variables** |
| Environment (approvals) | GitHub **Environment** on a job |
| `terraform-installer.yaml` | `hashicorp/setup-terraform@v3` |
| Self-hosted `pool: ado-devops-pool` | `runs-on` input (single label or JSON label array; default `ubuntu-latest`) |
| `##vso[task.setvariable ...;isOutput=true]` | `>> $GITHUB_OUTPUT` |
| `##vso[task.setvariable variable=PATH]` | `>> $GITHUB_PATH` |
| `##vso[task.logissue type=error]` | `::error::` workflow command |
| `$(Pipeline.Workspace)` / `$(Build.SourcesDirectory)` | `${{ github.workspace }}` |
| `$(Agent.TempDirectory)` | `${{ runner.temp }}` |
| `$(Build.SourceVersion)` | `${{ github.sha }}` |

## Authentication (OIDC)

All Azure interactions use **OIDC / workload identity federation** — no client secrets. Callers must:

1. Grant the token permission at the workflow (or job) level:
   ```yaml
   permissions:
     id-token: write
     contents: read
   ```
2. Configure a [federated credential](https://learn.microsoft.com/azure/developer/github/connect-from-azure)
   on an Azure AD app registration (or user-assigned managed identity) for this repository/environment.
3. Provide the identity via secrets (see each workflow's `secrets:` block).

Terraform authenticates through the `azurerm`/`azapi` providers using `ARM_USE_OIDC=true` plus
`ARM_CLIENT_ID` / `ARM_TENANT_ID` / `ARM_SUBSCRIPTION_ID`. The APIOps and API Center actions obtain a
bearer token via `az account get-access-token` after `azure/login`.

## Reusable workflows

### `terraform.yml` — Plan + Apply
Two jobs: `plan` (checkout → setup-terraform → optional vending record → init → workspace → plan →
upload short-lived `tfplan` artifact) and `apply` (checkout → download plan artifact → init → workspace → apply). The plan and apply
jobs can use **separate identities** (`azure-plan-client-id` / `azure-apply-client-id`) for least privilege;
set both to the same value if you only have one.

Key inputs: `module-path`, `terraform-version`, `terraform-workspace`, `terraform-action` (`apply`/`destroy`),
`override-vars` (JSON string), `plan-environment`, `apply-environment`, `subscription-vending-run`,
`table-*`, `partition-key`, `row-key`, `additional-var-files-*`, `show-plan-output`, `runs-on`.

### `ci.yml` — Validation
`validate` job (fmt check, `init -backend=false`, validate) plus a gated `plan` job. Set
`show-plan-output: true` only when the plan is safe to print to logs.

### `dotnet.yml`
Installs the SDK, builds, publishes, and uploads the output artifact.

### `api-ops.yml` — APIOps toolkit (Azure/apiops)
`plan` job runs the publisher in dry-run mode; `deploy` job applies. Requires Azure/apiops v7.0.0+.

### `api-ops-cli.yml` — APIOps CLI (Azure/apiops-cli)
Node.js-based equivalent. `plan` runs `apiops publish --dry-run`; `deploy` runs `apiops publish`.
Set `apiops-api-version` to override the Azure API Management REST API version used by the CLI; it defaults to `2024-05-01`.

### `api-center.yml` — Azure API Center
`validate` job (OpenAPI/Swagger + optional Spectral lint); `register` job (create + import to API Center).

## Wiring dependencies (the `projects[]` array)

Azure DevOps expanded a `projects` array with a `dependencies` list into stages at compile time.
GitHub Actions cannot dynamically generate jobs from an arbitrary array, so express the same dependency
graph explicitly with `needs:`. See [example-cd.yml](examples/example-cd.yml):

```yaml
jobs:
  lz-shared:                       # terraform
    uses: Remo-L5/github-action-templates/.github/workflows/terraform.yml@main
    # ...
  function-dev:                    # dotnet, depends on lz-shared
    needs: [lz-shared]
    uses: Remo-L5/github-action-templates/.github/workflows/dotnet.yml@main
  publisher-dev:                   # api-ops-cli, depends on function-dev
    needs: [function-dev]
    uses: Remo-L5/github-action-templates/.github/workflows/api-ops-cli.yml@main
  apicenter-dev:                   # api-center, depends on publisher-dev
    needs: [publisher-dev]
    uses: Remo-L5/github-action-templates/.github/workflows/api-center.yml@main
```

## Choosing the runner

Every reusable workflow accepts a `runs-on` input (single label, default `ubuntu-latest`, or a JSON label
array such as `["self-hosted","linux","x64"]`). The example
callers expose it once at the entry point: a `runs-on` `workflow_dispatch` input feeds a tiny `config` job,
whose output is passed to every downstream job via `needs.config.outputs.runs-on`. To target self-hosted
runners, set the dispatch input (for manual runs) or change the single fallback in the `config` job:

```yaml
on:
  workflow_dispatch:
    inputs:
      runs-on:
        type: string
        default: 'ubuntu-latest'   # e.g. ["self-hosted","linux","x64"]

jobs:
  config:
    runs-on: ${{ startsWith(inputs.runs-on || 'ubuntu-latest', '[') && fromJSON(inputs.runs-on || 'ubuntu-latest') || inputs.runs-on || 'ubuntu-latest' }}
    outputs:
      runs-on: ${{ inputs.runs-on || 'ubuntu-latest' }}
    steps:
      - run: echo "Runner: ${{ inputs.runs-on || 'ubuntu-latest' }}"

  lz-shared:
    needs: [config]
    uses: Remo-L5/github-action-templates/.github/workflows/terraform.yml@main
    with:
      runs-on: ${{ needs.config.outputs.runs-on }}
    # ...
```

## Prerequisites

- Azure AD app registration (or managed identity) with a **federated credential** for this repo/environment.
- **Secrets** (repository or environment scope):
  - `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`
  - `AZURE_PLAN_CLIENT_ID`, `AZURE_APPLY_CLIENT_ID` (Terraform), `AZURE_CLIENT_ID` (APIOps / API Center)
  - `BACKEND_AZURE_RESOURCE_GROUP_NAME`, `BACKEND_AZURE_STORAGE_ACCOUNT_NAME`, `BACKEND_AZURE_STORAGE_ACCOUNT_CONTAINER_NAME`
- **Environments** (for approvals): e.g. `alz-mgmt-plan`, `alz-mgmt-apply`, `apiops-plan`, `apiops-deploy`,
  `apicenter-validate`, `apicenter-register`.

> The lint warning *"Context access might be invalid: AZURE_TENANT_ID"* in the example callers simply means
> the secret isn't defined in your repository yet. It disappears once you add the secrets.

## Using these templates from another repository

Copy the example callers from `examples/` into a consuming repository's `.github/workflows/` directory and
reference reusable workflows by `owner/repo/.github/workflows/<file>@ref`:

```yaml
jobs:
  infra:
    uses: your-org/github-action-templates/.github/workflows/terraform.yml@main
    # ...
```

Composite actions are referenced with fully qualified `Remo-L5/github-action-templates/.github/actions/...@main`
paths inside the reusable workflows. This avoids depending on the caller repository having a matching
`.github/actions` folder after `actions/checkout`.

## Security and leakage tracking

Terraform plan files and generated `.auto.tfvars` files can contain environment-specific configuration and,
depending on module design, sensitive values. The Terraform deployment workflow now checks out the caller
repository in the apply job and uploads only the generated `tfplan` artifact with one-day retention instead
of uploading the full module directory. Subscription-vending records and generated HCL files are no longer
printed to logs, and Terraform plan output is hidden unless `show-plan-output` is explicitly enabled.

Track suspected template hardening gaps as normal GitHub issues labeled `security-hardening`. If you confirm
that a secret or sensitive value was exposed in logs, artifacts, git history, or a public issue, use a private
GitHub Security Advisory or private incident channel instead, rotate the affected credential, and delete or
expire affected workflow artifacts.

## License

MIT — see [LICENSE](LICENSE).