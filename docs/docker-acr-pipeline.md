### Docker ACR workflow inputs

| Input | Required | Default | Notes |
|-------|----------|---------|-------|
| `context` | no | `.` | Docker build context. |
| `dockerfile` | no | `Dockerfile` | Path to the Dockerfile. |
| `image_name` | yes | – | Repository name inside ACR (for example `api`). |
| `registry_name` | yes | – | ACR resource name (without `.azurecr.io`). |
| `semantic_version` | no | – | When provided, publishes `major`, `major.minor`, and full version tags. |
| `push_image` | no | `false` | Set to `true` when the build should push to ACR (for example on main merges). |
| `vulnerability_threshold` | no | `HIGH,CRITICAL` | Severities that cause the Trivy scan to fail. |

Required secret:

- `azure_credentials` – JSON output of `az ad sp create-for-rbac` (with `AcrPush`), used by `azure/login`.

### Workflow sequence

1. Checkout repository code.
2. Log into Azure using the provided service principal JSON.
3. If `push_image` is true, authenticate Docker to ACR with `az acr login`.
4. Set up QEMU and Docker Buildx for multi-architecture builds.
5. Run Trivy filesystem scan across the build context (fail on configured severities).
6. Calculate image tags with `docker/metadata-action@v5`.
7. Build (and optionally push) the image with BuildKit (`docker/build-push-action@v5`).

### Tagging strategy

- Always publishes branch and PR reference tags (`<branch>`, `pr-<number>`).
- Publishes a long SHA tag.
- When `semantic_version` is supplied (for example `1.4.2` or `v1.4.2`), the workflow adds:
  - Full version (`1.4.2`)
  - Major (`1`)
  - Major.minor (`1.4`)

### Recommendations

- Trigger on both `push` and `pull_request`. In PR builds keep `push: false` by calling the workflow with a wrapper job and passing `semantic_version` only on the main branch.
- Pair with `environment` protection or branch protection rules to guard production pushes.
- Consider adding `--build-arg` inputs to pass metadata (build date, commit hash) into the image if desired.
