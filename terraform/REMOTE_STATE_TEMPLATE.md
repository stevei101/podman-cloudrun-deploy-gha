# Terraform Cloud Remote State Template

Use this template to structure Terraform configurations that the reusable workflow can execute. Replace placeholder values with your organization's details.

```hcl
terraform {
  required_version = ">= 1.5.0"

  cloud {
    organization = "<YOUR_TERRAFORM_CLOUD_ORG>"

    workspaces {
      name = var.terraform_workspace_name
    }
  }
}

variable "terraform_workspace_name" {
  description = "Workspace name (e.g. dev, staging, prod)."
  type        = string
}

variable "gcp_project_id" {
  description = "Google Cloud project where resources are provisioned."
  type        = string
}

variable "region" {
  description = "Google Cloud region for Cloud Run and Artifact Registry."
  type        = string
}

variable "service_name" {
  description = "Cloud Run service name."
  type        = string
}

provider "google" {
  project = var.gcp_project_id
  region  = var.region
}

resource "google_artifact_registry_repository" "service" {
  location      = var.region
  repository_id = "${var.service_name}-repo"
  format        = "DOCKER"
}

resource "google_run_service" "service" {
  name     = var.service_name
  location = var.region

  template {
    spec {
      containers {
        image = "${var.region}-docker.pkg.dev/${var.gcp_project_id}/${google_artifact_registry_repository.service.repository_id}/${var.service_name}:latest"
        ports {
          name           = "http1"
          container_port = 8080
        }
        env {
          name  = "PORT"
          value = "8080"
        }
      }
    }
  }

  traffics {
    percent         = 100
    latest_revision = true
  }
}
```

## Variable injection from GitHub Actions

Populate Terraform variables via environment variables or a `*.tfvars` file. For example, create `terraform.auto.tfvars` in your service repository:

```hcl
gcp_project_id              = env.GCP_PROJECT_ID
region                      = env.CLOUD_RUN_REGION
service_name                = env.SERVICE_NAME
terraform_workspace_name    = env.TERRAFORM_WORKSPACE
```

The GitHub Actions workflow exports these variables automatically. Adjust naming to match your organization's conventions.

## Additional recommendations

- Configure IAM (`google_service_account`, `google_project_iam_member`) to grant Cloud Run runtime permissions.
- Add Firestore, Secret Manager, or other service resources as required.
- Use separate workspaces per environment (e.g., `dev`, `staging`, `prod`).
- Implement [run tasks](https://developer.hashicorp.com/terraform/cloud-docs/run-tasks) for security or policy enforcement.
