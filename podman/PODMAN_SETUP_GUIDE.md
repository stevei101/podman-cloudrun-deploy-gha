# Podman Setup Guide

This guide provides local development tips for teams adopting the Podman + Cloud Run pattern.

## 1. Install Podman

### macOS (Homebrew)

```bash
brew install podman
podman machine init
podman machine start
```

### Linux (Fedora, CentOS, RHEL)

```bash
sudo dnf install -y podman
```

### Windows (WSL2)

Use the Windows installer from the [Podman releases](https://github.com/containers/podman/releases) page or install within WSL2 using your distribution's package manager.

## 2. Configure Docker compatibility (optional)

Set the `DOCKER_HOST` environment variable if you want Docker-compatible tooling to talk to Podman:

```bash
export DOCKER_HOST=unix:///run/user/$(id -u)/podman/podman.sock
```

macOS users can enable the Docker socket:

```bash
podman machine ssh -- systemctl --user enable --now podman.socket
```

## 3. Authenticate to Artifact Registry locally

1. Ensure `gcloud` is installed and authenticated (`gcloud auth login`).
2. Obtain an access token and login with Podman:

```bash
REGION="us-central1"
PROJECT_ID="your-project-id"
ACCESS_TOKEN=$(gcloud auth print-access-token)
podman login "https://${REGION}-docker.pkg.dev" \
  --username oauth2accesstoken \
  --password "${ACCESS_TOKEN}"
```

## 4. Build and push locally

Reuse the same naming pattern as the GitHub Actions workflow:

```bash
SERVICE_NAME="sample-service"
GAR_REPO="services"
TAG="dev"
REGION="us-central1"
PROJECT_ID="your-project-id"

IMAGE_URI="${REGION}-docker.pkg.dev/${PROJECT_ID}/${GAR_REPO}/${SERVICE_NAME}:${TAG}"

podman build -t "${IMAGE_URI}" -f Dockerfile .
podman push "${IMAGE_URI}"
```

## 5. Emulator ports and Cloud Run compatibility

- Bind web servers to `0.0.0.0` so they accept traffic on the port forwarded by Podman.
- Read the `PORT` environment variable in your application code; Cloud Run sets it automatically.

## 6. Troubleshooting

| Symptom | Fix |
| --- | --- |
| `Error: cannot connect to Podman socket` | Ensure `podman machine` is running (`podman machine start`). |
| Authentication errors when pushing to GAR | Re-run the `podman login` command after refreshing GCP credentials. |
| Slow macOS builds | Increase Podman VM resources: `podman machine stop && podman machine init --cpus=4 --memory=8192 && podman machine start`. |

Refer to the official [Podman documentation](https://podman.io/docs) for additional scenarios.
