# Configuration Guide

This guide describes the configuration surface area for the `podman-cloudrun-deploy-gha` pattern repository.

## 1. Workflow inputs

| Input | Required | Default | Purpose |
| --- | --- | --- | --- |
| `environment` | ✅ | — | Logical deployment environment (shown in logs) |
| `service_name` | ✅ | — | Cloud Run service to deploy |
| `cloud_run_region` | ✅ | — | Cloud Run region (e.g. `us-central1`, `europe-west1`) |
| `gar_repository` | ✅ | — | Artifact Registry repository name |
| `dockerfile` | ❌ | `Dockerfile` | Dockerfile path relative to repo root |
| `build_context` | ❌ | `.` | Build context directory |
| `image_tag` | ❌ | `${{ github.sha }}` | Image tag applied to the build |
| `container_port` | ❌ | `8080` | Container port. Added to `--set-env-vars PORT=<value>` |
| `allow_unauthenticated` | ❌ | `false` | If `true`, deploys with `--allow-unauthenticated` |
| `additional_env_vars` | ❌ | `""` | Extra `KEY=VALUE` pairs (comma separated) passed to `--set-env-vars` |
| `additional_secrets` | ❌ | `""` | `SECRET_NAME=SECRET_ID:version` pairs for `--set-secrets` |
| `terraform_directory` | ❌ | `""` | Terraform folder. Leave empty to skip Terraform |
| `terraform_workspace` | ❌ | `""` | Workspace to select or create when Terraform runs |
| `terraform_version` | ❌ | `1.7.5` | Terraform CLI version |

### Example variable strategy

Define repository variables reflecting each environment:

| Variable | Value (staging) | Value (production) |
| --- | --- | --- |
| `CLOUD_RUN_SERVICE_NAME` | `agentnav-backend-stg` | `agentnav-backend` |
| `CLOUD_RUN_REGION` | `europe-west1` | `europe-west1` |
| `GAR_REPOSITORY` | `services-stg` | `services` |

Use workflow inputs to map these variables when calling the reusable workflow.

## 2. Secrets

| Secret | Required | Notes |
| --- | --- | --- |
| `GCP_PROJECT_ID` | ✅ | Must match the project hosting Cloud Run & Artifact Registry |
| `WIF_PROVIDER` | ✅ | Format: `projects/<project-number>/locations/global/workloadIdentityPools/<pool-name>/providers/<provider-name>` |
| `WIF_SERVICE_ACCOUNT` | ✅ | Service account with `roles/run.admin`, `roles/artifactregistry.writer`, `roles/iam.serviceAccountTokenCreator`, and Terraform-related roles (if Terraform runs) |
| `TF_API_TOKEN` | ⚠️ | Required only when `terraform_directory` is non-empty |

### Terraform Cloud permissions

- Token scope: user/team with access to the configured workspace.
- The workspace must use remote execution or a compatible backend.
- Update the Terraform service account roles (`roles/run.admin`, `roles/artifactregistry.writer`, `roles/iam.serviceAccountUser`, etc.) to align with your Infrastructure as Code plan.

## 3. Artifact Registry naming

Artifact Registry follows the pattern `REGION-docker.pkg.dev/PROJECT/REPOSITORY/IMAGE:TAG`.

Map the workflow inputs accordingly:

- `cloud_run_region` → `REGION`
- `GCP_PROJECT_ID` → `PROJECT`
- `gar_repository` → `REPOSITORY`
- `service_name` → `IMAGE`
- `image_tag` → `TAG`

The workflow automatically publishes both `${image_tag}` and `latest` tags. Delete the `podman tag ... latest` line if you prefer immutable tags only.

## 4. Terraform Cloud integration

To enable the Terraform step:

1. Place your Terraform code in the path referenced by `terraform_directory`.
2. Configure `terraform { cloud { organization = "<ORG>" workspaces { name = "<WORKSPACE>" } } }` or an equivalent `backend "remote"` block.
3. Provide a `TF_API_TOKEN` secret.
4. Set `terraform_workspace` so the workflow can select/create it on the fly.

When the workflow runs it will:

1. Write a `~/.terraformrc` file containing your API token.
2. Initialize the configuration `terraform -chdir=<dir> init -input=false`.
3. Select or create the workspace.
4. Apply with `-auto-approve`.

> **Note:** If your organization mandates manual approvals, replace the `apply` command with a `plan` step and integrate Terraform Cloud run tasks or approvals accordingly.

## 5. Cloud Run settings

- `container_port` should align with your container's exposed port.
- Cloud Run automatically sets the `PORT` environment variable; the workflow injects the same value explicitly for compatibility with app frameworks that expect it.
- Use `additional_env_vars` for values that do not belong in secrets (e.g., `ENVIRONMENT`, `FIRESTORE_PROJECT_ID`).
- Use `additional_secrets` to bind secrets stored in Secret Manager (format `VAR_NAME=SECRET_ID:version`).

## 6. Podman notes

- The workflow uses `containers/podman-action@v2` to provision Podman on the GitHub-hosted runner.
- Local development guidance is provided in `podman/PODMAN_SETUP_GUIDE.md`.
- Ensure your Dockerfile uses relative copy paths compatible with the repository root.

## 7. Extending the pattern

Consider augmenting the workflow with any of the following based on your service needs:

- matrix strategy for multi-service deployments (frontend + backend)
- vulnerability scans (`trivy`, `grype`) before pushing images
- integration tests before deployment
- `gcloud run services describe` output formatting for Slack notifications
- GitHub Environments for manual approvals and secrets scoping

Keep documentation updated when you extend the pattern to maintain consistency across `stevei101` repositories.
