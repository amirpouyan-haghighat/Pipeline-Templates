### Terraform workflow inputs

| Input                 | Required | Default      | Notes                                                                                  |
| --------------------- | -------- | ------------ | -------------------------------------------------------------------------------------- |
| `terraform_directory` | yes      | –            | Relative path to the Terraform configuration.                                          |
| `terraform_version`   | no       | `1.6.6`      | Override to pin the CLI version used.                                                  |
| `pr_number`           | no       | –            | Needed only when you want the plan posted as a PR comment.                             |
| ` args`               | no       | –            | Extra arguments appended to `terraform plan`.                                          |
| ` args`               | no       | –            | Extra arguments appended to `terraform apply`.                                         |
| `run_apply`           | no       | `false`      | Set to `true` to enable apply.                                                         |
| `environment_name`    | no       | `production` | The GitHub environment used to gate the apply job. Configure required reviewers there. |
| `backend_config_file` | no       | –            | Passed to `terraform init -backend-config`.                                            |

Optional secret:

- `terraform_cloud_token` – token for Terraform Cloud/Enterprise CLI auth (`cli_config_credentials_token`).

### Terraform workflow sequence

1. Checkout repository code.
2. Install Terraform via `hashicorp/setup-terraform@v2`.
3. Run `terraform fmt -check` to enforce formatting.
4. Run `terraform init` (optionally with backend config).
5. Run `terraform plan `, capture the log, and upload it as an artifact.
6. If a PR number is supplied, comment the tail of the plan directly on the PR.
7. When `run_apply` is true, repeat init/plan and execute `terraform apply -auto-approve` behind an environment gate.

### Manual approvals

- Configure the GitHub environment named in `environment_name` (defaults to `production`).
- In repository settings → Environments, add required reviewers to demand manual approval before the apply job runs.

### Recommendations

- Call the workflow from `pull_request` and `push` (`main`) events separately so PRs never invoke apply.
- Store state remotely (Terraform Cloud, Azure Storage, etc.) and pass relevant backend files using `backend_config_file`.
