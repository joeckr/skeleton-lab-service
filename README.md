# skeleton-lab-service

A starter skeleton repository tailored for rapid experimentation, prototyping, and testing of services in a lab environment (local Docker, Kubernetes, and OpenShift).

> **Note**: This repository intentionally **does NOT include a Dockerfile**. If a custom image build is required, use an OCI image repository (such as `skeleton-oci-modified`). Lab repositories focus on configuring, deploying, and testing upstream or pre-built container services.

---

## Features

- **Helm Chart (`chart/`)**:
  - Out-of-the-box support for deploying services to Kubernetes and OpenShift.
  - Parameterized upstream container image (`image:tag`), replica count, and ports.
  - Dual support for standard Kubernetes Ingress and OpenShift Routes (`ingress.route: "true"`).
  - Secure defaults: non-root execution (`runAsNonRoot: true`), `RuntimeDefault` seccomp profile, and dropping `ALL` capabilities.
  - Master and control-plane node tolerations for compact lab clusters.
- **Docker Compose (`docker-compose.yml`)**:
  - Spin up and test the lab service locally in seconds without requiring a cluster.
  - Pre-configured with port forwarding, and healthchecks.
- **Environment & Tooling (`mise` & `prek`)**:
  - `mise.toml`: Tool version management (`helm`, `gitleaks`, `addlicense`, `trivy`, `actionlint`) and convenient task aliases.
  - `prek.toml`: Fast git hooks enforcing Conventional Commits, branch protection, secrets scanning, Helm linting, workflow linting (`actionlint`), and security audits (`zizmor`).
- **GitHub Actions CI (`.github/workflows/`)**:
  - Reusable workflows powered by [`joeckr/ci-templates`](https://github.com/joeckr/ci-templates):
    - `actionlint`: Lints GitHub Actions workflow syntax.
    - `zizmor`: Security audit of GitHub Actions workflows.
    - `commitlint`: Enforces Conventional Commits specification.
    - `gitleaks`: Scans commits and PRs for secret leaks.
    - `helm`: Packages and publishes Helm charts to GitHub Container Registry (GHCR) as OCI artifacts (with PR dry-run preview).
    - `semantic`: Automated Semantic Versioning, git tagging, release notes, and `Chart.yaml` version syncing (with PR dry-run preview).

---

## Directory Structure

```text
.
├── .github/
│   └── workflows/
│       ├── actionlint.yml       # Lints workflow files
│       ├── commitlint.yml       # Validates conventional commit messages
│       ├── gitleaks.yml         # Scans for credential leaks
│       ├── helm.yml             # Packages and pushes Helm chart to GHCR
│       ├── semantic.yml         # SemVer tagging and GitHub releases
│       ├── test_helm.yml        # PR dry-run test for Helm packaging
│       ├── test_semantic.yml    # PR dry-run test for Semantic Versioning
│       └── zizmor.yml           # Security audit for workflows
├── chart/
│   ├── Chart.yaml               # Helm chart definition
│   ├── values.yaml              # Default configuration values
│   ├── .helmignore              # Ignore rules for chart packaging
│   ├── config/
│   │   └── config.txt           # Config file mounted into containers
│   └── templates/
│       ├── deployment.yaml      # Workload deployment
│       ├── service.yaml         # Kubernetes Service
│       ├── ingress.yaml         # Kubernetes Ingress
│       ├── route.yaml           # OpenShift Route
│       ├── pvc.yaml             # PersistentVolumeClaim
│       └── mount-config-map.yaml# ConfigMap resource
├── config/
│   └── config.txt               # Local configuration file for docker compose
├── scripts/
│   ├── setup.sh                 # Environment check and hook installation
│   ├── lab-up.sh                # Start local compose service
│   ├── lab-down.sh              # Stop local compose service
│   ├── helm-lint.sh             # Lint chart
│   ├── helm-template.sh         # Render chart templates locally
│   ├── install.sh               # Install/upgrade chart into a cluster
│   └── uninstall.sh             # Uninstall release from cluster
├── docker-compose.yml           # Local lab service definition
├── mise.toml                    # Mise tools and tasks
├── prek.toml                    # Prek git hooks
└── README.md
```

---

## Quickstart

### 1. Bootstrap Local Environment

Ensure [`mise`](https://mise.jdx.dev/) and [`prek`](https://github.com/j178/prek) are installed:

```bash
# Verify environment and install git hooks
./scripts/setup.sh

# Or using mise directly
mise run install
```

### 2. Local Experimentation (Docker Compose)

Start the lab service locally:

```bash
# Start container in detached mode
./scripts/lab-up.sh
# or: mise run compose

# Check status and logs
docker compose ps
mise run logs

# Stop container
./scripts/lab-down.sh
# or: mise run down

# Stop container and clean test volumes
./scripts/lab-down.sh -v
```

By default, the service listens at `http://localhost:8080`.

### 3. Kubernetes / OpenShift Experimentation (Helm)

#### Lint and Template Locally
```bash
# Lint the chart
./scripts/helm-lint.sh
# or: mise run helm-lint

# Render manifests to stdout
./scripts/helm-template.sh
# or: mise run helm-template

# Test OpenShift Route rendering
helm template lab-service chart/ --set ingress.enabled=true --set ingress.route=true
```

#### Deploy to a Cluster
```bash
# Deploy to namespace 'lab-service'
./scripts/install.sh

# Deploy with custom namespace and values
NAMESPACE=my-test ./scripts/install.sh --set app.image=quay.io/my/service --set app.tag=v1.0.0

# Preview changes with dry-run
./scripts/install.sh --dry-run

# Uninstall
./scripts/uninstall.sh
```

---

## Conventional Commits & Releases

Commits must follow the [Conventional Commits](https://www.conventionalcommits.org/) specification:
- `feat: add new feature` -> Triggers a **minor** release (e.g., `v0.1.0` -> `v0.2.0`).
- `fix: resolve issue` -> Triggers a **patch** release (e.g., `v0.1.0` -> `v0.1.1`).
- `feat!: breaking change` -> Triggers a **major** release.
- `chore:`, `docs:`, `ci:`, `test:`, `refactor:` -> Maintenance changes (no release bump).

Upon merging to `main`, the `semantic.yml` workflow automatically computes the next version, updates `version` and `appVersion` in `chart/Chart.yaml`, creates a Git tag, and publishes a GitHub Release. The `helm.yml` workflow packages the chart and pushes it to GHCR.
