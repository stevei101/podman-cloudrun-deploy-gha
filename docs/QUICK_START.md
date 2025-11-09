# Quick Start

This quick start walks through wiring the reusable Podman → Terraform → Cloud Run pipeline into an existing service repository.

## 1. Fork or copy the pattern repository

Clone this repository or copy the `.github/workflows/podman-cloudrun-deploy.yaml` file into your project. Keep the documentation nearby for reference while you configure secrets.

## 2. Configure GitHub secrets and variables

Add the following repository secrets (Settings → Secrets and variables → Actions):

| Name | Example | Description |
| --- | --- | --- |
| `GCP_PROJECT_ID` | `prod-agentnav-123456` | Google Cloud project ID |
| `WIF_PROVIDER` | `projects/1234567890/locations/global/workloadIdentityPools/github/providers/gha` | Workload Identity Federation provider resource |
| `WIF_SERVICE_ACCOUNT` | `gha-deployer@prod-agentnav-123456.iam.gserviceaccount.com` | Service account email impersonated by the workflow |
| `TF_API_TOKEN` | (optional) | Terraform Cloud API token if you run Terraform from the workflow |

For reusable values (service name, region, repository name) create **repository variables** (Settings → Secrets and variables → Actions → Variables). Example:

| Name | Value |
| --- | --- |
| `CLOUD_RUN_SERVICE_NAME` | `agentnav-backend` |
| `CLOUD_RUN_REGION` | `europe-west1` |
| `GAR_REPOSITORY` | `services` |
| `DEPLOY_ENV` | `staging` |

## 3. Reference the reusable workflow

Create a workflow in your service repo (e.g. `.github/workflows/deploy-staging.yaml`) that calls the reusable workflow:

```yaml
name: Deploy to staging

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    uses: stevei101/podman-cloudrun-deploy-gha/.github/workflows/podman-cloudrun-deploy.yaml@main
    with:
      environment: staging
      service_name: ${{ vars.CLOUD_RUN_SERVICE_NAME }}
      cloud_run_region: ${{ vars.CLOUD_RUN_REGION }}
      gar_repository: ${{ vars.GAR_REPOSITORY }}
      dockerfile: Dockerfile
      build_context: .
      container_port: "8080"
      allow_unauthenticated: false
      terraform_directory: infrastructure/terraform
      terraform_workspace: staging
    secrets:
      GCP_PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
      WIF_PROVIDER: ${{ secrets.WIF_PROVIDER }}
      WIF_SERVICE_ACCOUNT: ${{ secrets.WIF_SERVICE_ACCOUNT }}
      TF_API_TOKEN: ${{ secrets.TF_API_TOKEN }}
```

Skip Terraform by setting `terraform_directory: ""`.

## 4. Verify Terraform configuration (optional)

- Ensure your Terraform code targets Terraform Cloud (remote backend or `cloud` block).
- The workflow selects/creates the workspace named by `terraform_workspace`.
- The `TF_API_TOKEN` secret must belong to a Terraform Cloud user/team with access to the workspace.

## 5. Confirm Podman build viability

Before relying on GitHub Actions, build locally with Podman using the same `Dockerfile` and context:

```bash
podman build -t test-build:dev -f Dockerfile .
```

If the build succeeds locally, the workflow should succeed as well.

## 6. Run the workflow

Push a commit to the branch monitored by your trigger (e.g. `main`). The workflow will:

1. Authenticate using WIF
2. Initialize and apply Terraform (if configured)
3. Build and tag the container image with Podman
4. Push the image to Artifact Registry
5. Deploy to Cloud Run with shared `PORT` configuration

Check the GitHub Actions logs and Cloud Run console for confirmation. Update variables or inputs as your environments evolve.
