## Pipeline Templates

Reusable GitHub Actions workflows for Terraform deployments and Azure Container Registry (ACR) image publishing. Other repositories in your organisation can reference these workflows with `workflow_call` to share one centralised automation implementation.

### Repository structure

- `.github/workflows/terraform-reusable.yml` – Terraform plan/apply workflow.
- `.github/workflows/docker-acr-reusable.yml` – Docker build/scan/push workflow for ACR.
- `docs/` – Additional notes and examples.
- `terraform/` – Placeholder for any shared Terraform helpers or modules you may add later.

### Terraform workflow

The workflow can be invoked from another repository:

```yaml
name: terraform

on:
  pull_request:
  push:
    branches: [main]

jobs:
  plan:
    if: github.event_name == 'pull_request'
    uses: your-org/Pipeline-Templates/.github/workflows/terraform-reusable.yml@main
    with:
      terraform_directory: infra/environments/dev
      pr_number: ${{ github.event.pull_request.number }}
    secrets:
      terraform_cloud_token: ${{ secrets.TF_API_TOKEN }}

  apply:
    if: github.event_name == 'push' && github.ref_name == 'main'
    uses: your-org/Pipeline-Templates/.github/workflows/terraform-reusable.yml@main
    with:
      terraform_directory: infra/environments/prod
      run_apply: true
      environment_name: production
    secrets:
      terraform_cloud_token: ${{ secrets.TF_API_TOKEN }}
```

Key behaviour:

- Pull requests run `terraform fmt -check`, `terraform init`, and `terraform plan`. The final 200 lines of the plan are posted as a PR comment and the full log is stored as an artifact.
- Merge (or any `run_apply: true` call) repeats init/plan and then runs `terraform apply -auto-approve`. The job is bound to the named environment so you can require manual reviewers before apply runs.
- Optional inputs let you pass extra CLI flags or a backend configuration file.

### Docker ACR workflow

Example invocation:

```yaml
name: docker

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build-and-push:
    uses: your-org/Pipeline-Templates/.github/workflows/docker-acr-reusable.yml@main
    with:
      context: services/api
      dockerfile: services/api/Dockerfile
      image_name: api
      registry_name: myacr
      semantic_version: ${{ github.ref_name }}
      push_image: ${{ github.event_name == 'push' && github.ref_name == 'main' }}
    secrets:
      azure_credentials: ${{ secrets.AZURE_CREDENTIALS }}
```

Key behaviour:

- Logs into Azure using a service principal, then authenticates to the given ACR.
- Runs a Trivy filesystem scan across the build context (Dockerfile, dependencies, secrets) and fails the workflow on high/critical issues by default.
- Builds and pushes the image with BuildKit/Buildx. Tags include branch and SHA; when `semantic_version` is provided, `major`, `major.minor`, and full version tags are published automatically.

### GitHub Actions syntax highlights

- `on.workflow_call` makes a workflow reusable. Callers can supply `inputs` (strongly typed) and `secrets` that surface inside the workflow.
- `permissions` scope the default `GITHUB_TOKEN`. Least privilege is applied (`pull-requests: write` is needed for PR comments).
- `env` defines environment variables shared by all jobs/steps (e.g., `TF_IN_AUTOMATION`).
- `jobs.<id>.if` enables conditional execution (apply job runs only when `run_apply` is `true`).
- `needs` links jobs so `terraform_apply` waits for `terraform_plan`.
- `environment` binds a job to a named GitHub environment; configure required reviewers there for manual approval gates.
- `with` passes inputs to actions (like `hashicorp/setup-terraform@v2` or `docker/build-push-action@v5`).
- `run` commands execute in the runner shell; honour `working-directory` for Terraform commands.
- `${{ }}` expressions access inputs, secrets, and context (`github.*`).
- `actions/upload-artifact` exposes logs for download by storing artifacts generated during jobs.

See `docs/` for future expansions or tailored guidance.
