# Security

## Reporting suspected leakage

If a workflow template may expose sensitive configuration, track the hardening work in a normal GitHub issue
with a `security-hardening` label.

If a secret or sensitive value was actually exposed in logs, artifacts, git history, or a public issue, do not
track the details in a public issue. Use a private GitHub Security Advisory or private incident channel, rotate
the affected credential, and remove or expire affected workflow artifacts.

## Template guidance

- Prefer OIDC federation over long-lived client secrets.
- Keep Terraform plan artifacts short-lived. The provided Terraform workflow uses one-day artifact retention.
- Do not print generated `.tfvars`, storage table records, access tokens, or full Terraform plans containing
  unreviewed sensitive values to workflow logs. Leave `show-plan-output` disabled unless the plan is safe to print.
- Use GitHub Environments for deployment approvals and environment-scoped secrets.
