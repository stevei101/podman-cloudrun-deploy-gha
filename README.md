# Podman + Cloud Run Deploy with GitHub Actions

A generic, anonymized pattern repository that demonstrates how to:

- authenticate to Google Cloud via Workload Identity Federation (WIF)
- optionally run Terraform Cloud-backed infrastructure updates
- build OCI images with Podman
- push images to Google Artifact Registry (GAR)
- deploy services to Google Cloud Run with the correct `PORT` handling

Use this repository as a starting point for any service that follows the **Podman → Terraform → Cloud Run** pipeline.

## Repository Layout

```
podman-cloudrun-deploy-gha/
├── .github/
│   └── workflows/
│       └── podman-cloudrun-deploy.yaml   # Reusable workflow (workflow_call)
├── docs/
│   ├── CONFIGURATION_GUIDE.md            # Environment + secret setup
│   └── QUICK_START.md                    # One-page adoption guide
├── podman/
│   └── PODMAN_SETUP_GUIDE.md             # Local Podman guidance and tips
├── terraform/
│   └── REMOTE_STATE_TEMPLATE.md          # Terraform Cloud workspace pattern
└── README.md
```

## Supported Workflow Inputs

The reusable workflow (`podman-cloudrun-deploy.yaml`) is designed to be invoked via `workflow_call`. A typical consumer workflow looks like this:

```yaml
name: Deploy (staging)

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
      dockerfile: infrastructure/docker/Dockerfile
      build_context: infrastructure/docker
      image_tag: ${{ github.sha }}
      container_port: "8080"
      allow_unauthenticated: false
      additional_env_vars: "ENVIRONMENT=${{ vars.DEPLOY_ENV }},ADK_AGENT_CONFIG_PATH=agents/config.yaml"
      additional_secrets: "GEMINI_API_KEY=GEMINI_API_KEY:latest,FIRESTORE_KEY=FIRESTORE_SERVICE_ACCOUNT:latest"
      terraform_directory: infrastructure/terraform
      terraform_workspace: staging
    secrets:
      GCP_PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
      WIF_PROVIDER: ${{ secrets.WIF_PROVIDER }}
      WIF_SERVICE_ACCOUNT: ${{ secrets.WIF_SERVICE_ACCOUNT }}
      TF_API_TOKEN: ${{ secrets.TF_API_TOKEN }}
```

Set `terraform_directory` to an empty string (`""`) if you want to skip Terraform.

### Required Secrets

| Secret | Purpose |
| --- | --- |
| `GCP_PROJECT_ID` | Target Google Cloud project ID |
| `WIF_PROVIDER` | Fully-qualified Workload Identity Federation provider resource |
| `WIF_SERVICE_ACCOUNT` | Service account email to impersonate |

### Optional Secret

| Secret | Purpose |
| --- | --- |
| `TF_API_TOKEN` | Terraform Cloud API token (required if Terraform step is enabled) |

## Artifact Registry & Image Naming

Images follow the pattern:

```
${REGION}-docker.pkg.dev/${GCP_PROJECT_ID}/${GAR_REPOSITORY}/${SERVICE_NAME}:${IMAGE_TAG}
```

Alongside the commit-specific tag (`${IMAGE_TAG}`), the workflow also publishes a mutable `latest` tag for convenience. Update this behavior in the workflow if your team prefers immutable tags only.

## Cloud Run Deployment Flags

The deploy step always injects `PORT=${container_port}` to satisfy Cloud Run expectations. Use the `additional_env_vars` and `additional_secrets` inputs to include service-specific parameters without modifying the reusable workflow.

## Next Steps

1. Read the quick start in `docs/QUICK_START.md`.
2. Configure repository secrets and variables using `docs/CONFIGURATION_GUIDE.md`.
3. Reference the reusable workflow from your service repository.
4. Tailor local Podman setup via `podman/PODMAN_SETUP_GUIDE.md`.

Contributions that extend the pattern (e.g., support for multiple services, canary deployments, or additional verification steps) are welcome—submit a PR with documentation updates.
