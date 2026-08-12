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
├── workflows/                         # Reusable workflows (workflow_call) + example callers
│   ├── terraform.yml                  # Terraform plan + apply
│   ├── ci.yml                         # Terraform fmt/validate + review plan (PRs)
│   ├── dotnet.yml                     # Build + publish .NET apps
│   ├── api-ops.yml                    # APIOps toolkit plan/deploy
│   ├── api-ops-cli.yml                # APIOps CLI plan/deploy
│   ├── api-center.yml                 # API Center validate/register
│   ├── example-cd.yml                 # Example CD caller wiring multiple projects
│   └── example-ci.yml                 # Example CI caller
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
| Self-hosted `pool: ado-devops-pool` | `runs-on` input (single label; default `ubuntu-latest`) |
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
upload `module` artifact) and `apply` (download artifact → init → workspace → apply). The plan and apply
jobs can use **separate identities** (`azure-plan-client-id` / `azure-apply-client-id`) for least privilege;
set both to the same value if you only have one.

Key inputs: `module-path`, `terraform-version`, `terraform-workspace`, `terraform-action` (`apply`/`destroy`),
`override-vars` (JSON string), `plan-environment`, `apply-environment`, `subscription-vending-run`,
`table-*`, `partition-key`, `row-key`, `additional-var-files-*`, `runs-on`.

### `ci.yml` — Validation
`validate` job (fmt check, `init -backend=false`, validate) plus a gated `plan` job for review.

### `dotnet.yml`
Installs the SDK, builds, publishes, and uploads the output artifact.

### `api-ops.yml` — APIOps toolkit (Azure/apiops)
`plan` job runs the publisher in dry-run mode; `deploy` job applies. Requires Azure/apiops v7.0.0+.

### `api-ops-cli.yml` — APIOps CLI (Azure/apiops-cli)
Node.js-based equivalent. `plan` runs `apiops publish --dry-run`; `deploy` runs `apiops publish`.

### `api-center.yml` — Azure API Center
`validate` job (OpenAPI/Swagger + optional Spectral lint); `register` job (create + import to API Center).

## Wiring dependencies (the `projects[]` array)

Azure DevOps expanded a `projects` array with a `dependencies` list into stages at compile time.
GitHub Actions cannot dynamically generate jobs from an arbitrary array, so express the same dependency
graph explicitly with `needs:`. See [example-cd.yml](.github/workflows/example-cd.yml):

```yaml
jobs:
  lz-shared:                       # terraform
    uses: ./.github/workflows/terraform.yml
    # ...
  function-dev:                    # dotnet, depends on lz-shared
    needs: [lz-shared]
    uses: ./.github/workflows/dotnet.yml
  publisher-dev:                   # api-ops-cli, depends on function-dev
    needs: [function-dev]
    uses: ./.github/workflows/api-ops-cli.yml
  apicenter-dev:                   # api-center, depends on publisher-dev
    needs: [publisher-dev]
    uses: ./.github/workflows/api-center.yml
```

## Choosing the runner

Every reusable workflow accepts a `runs-on` input (single label, default `ubuntu-latest`). The example
callers expose it once at the entry point: a `runs-on` `workflow_dispatch` input feeds a tiny `config` job,
whose output is passed to every downstream job via `needs.config.outputs.runs-on`. To target self-hosted
runners, set the dispatch input (for manual runs) or change the single fallback in the `config` job:

```yaml
on:
  workflow_dispatch:
    inputs:
      runs-on:
        type: string
        default: 'ubuntu-latest'   # e.g. self-hosted

jobs:
  config:
    runs-on: ${{ inputs.runs-on || 'ubuntu-latest' }}
    outputs:
      runs-on: ${{ inputs.runs-on || 'ubuntu-latest' }}
    steps:
      - run: echo "Runner: ${{ inputs.runs-on || 'ubuntu-latest' }}"

  lz-shared:
    needs: [config]
    uses: ./.github/workflows/terraform.yml
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

The example callers reference the workflows with a local path (`./.github/workflows/...`). To consume them
from a **different** repository, reference them by `owner/repo/.github/workflows/<file>@ref`:

```yaml
jobs:
  infra:
    uses: your-org/github-action-templates/.github/workflows/terraform.yml@main
    # ...
```

Composite actions referenced as `./.github/actions/...` inside a reusable workflow resolve against the
repository that **defines** the workflow, so cross-repo consumption works when the actions live beside the
workflows in this repo.

## License

MIT — see [LICENSE](LICENSE).