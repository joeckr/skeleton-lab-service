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
- **Environment & Tooling (`mise` & `hk`)**:
  - `mise.toml`: Tool version management (`helm`, `betterleaks`, `addlicense`, `trivy`, `actionlint`, `shellcheck`, `zizmor`, `hk`, `pkl`, `tombi`, `yamllint`) and convenient task aliases.
  - `hk.pkl`: Fast git hooks and project checks enforcing Conventional Commits, branch protection, secret scanning (`betterleaks`), Helm linting, workflow linting (`actionlint`), security audits (`zizmor`), YAML linting (`yamllint`), TOML linting (`tombi`), shell checking (`shellcheck`), and license checks (`addlicense`).
- **GitHub Actions CI (`.github/workflows/`)**:
  - Reusable workflows powered by [`joeckr/ci-templates`](https://github.com/joeckr/ci-templates):
    - `lint.yml`: Lints GitHub Actions workflow syntax (`actionlint`), Conventional Commits (`commitlint`), and shell scripts (`shellcheck`).
    - `security.yml`: Scans commits and PRs for secret leaks (`betterleaks`) and workflow security audits (`zizmor`).
    - `release.yml`: Automated Semantic Versioning, git tagging, release notes, `Chart.yaml` version syncing (`semantic`), and packaging/publishing Helm charts to GitHub Container Registry (GHCR) as OCI artifacts.
    - `test_release.yml`: PR dry-run preview for Semantic Versioning and Helm chart packaging.

---

## Directory Structure

```text
.
├── .github/
│   └── workflows/
│       ├── lint.yml             # Lints workflows, commits, and shell scripts
│       ├── security.yml         # Scans for secrets and audits workflow security
│       ├── release.yml          # Semantic release and Helm chart publishing to GHCR
│       └── test_release.yml     # PR dry-run test for releases and chart packaging
├── chart/
│   ├── Chart.yaml               # Helm chart definition
│   ├── values.yaml              # Default configuration values
│   ├── .helmignore              # Ignore rules for chart packaging
│   └── templates/
│       ├── deployment.yaml      # Workload deployment
│       ├── service.yaml         # Kubernetes Service
│       ├── ingress.yaml         # Kubernetes Ingress
│       └── route.yaml           # OpenShift Route
├── config/
│   └── config.txt               # Local configuration file for docker compose
├── scripts/
│   └── template.sh              # Helper script placeholder
├── docker-compose.yml           # Local lab service definition
├── hk.pkl                       # hk git hooks and checks configuration
├── mise.toml                    # Mise tools and tasks
└── README.md
```

---

## Quickstart

### 1. Bootstrap Local Environment

Ensure [`mise`](https://mise.jdx.dev/) is installed, then prepare the environment and install git hooks:

```bash
mise run install
```

This installs all declared tools via `mise` and hooks up git hooks using `hk`.

To run all checks across the repository at any time:

```bash
mise run check
# or: mise run hk
```

### 2. Local Experimentation (Docker Compose)

Start the lab service locally:

```bash
# Start container in detached mode
mise run compose
# or: docker compose up -d

# Check status and logs
docker compose ps
mise run logs

# Stop container
mise run down
# or: docker compose down
```

By default, the service listens at `http://localhost:8080`.

### 3. Kubernetes / OpenShift Experimentation (Helm)

#### Lint and Template Locally
```bash
# Lint the chart
mise run helm-lint
# or: helm lint chart/

# Render manifests to stdout
mise run helm-template
# or: helm template lab-service chart/

# Test OpenShift Route rendering
helm template lab-service chart/ --set ingress.enabled=true --set ingress.route="true"
```

#### Deploy to a Cluster
```bash
# Deploy to namespace 'lab-service'
helm upgrade --install lab-service chart/ --namespace lab-service --create-namespace

# Deploy with custom namespace and values
helm upgrade --install lab-service chart/ --namespace my-test --create-namespace --set app.image=quay.io/my/service --set app.tag=v1.0.0

# Preview changes with dry-run
helm upgrade --install lab-service chart/ --namespace lab-service --dry-run
```

---

## Conventional Commits & Releases

Commits must follow the [Conventional Commits](https://www.conventionalcommits.org/) specification:
- `feat: add new feature` -> Triggers a **minor** release (e.g., `v0.1.0` -> `v0.2.0`).
- `fix: resolve issue` -> Triggers a **patch** release (e.g., `v0.1.0` -> `v0.1.1`).
- `feat!: breaking change` -> Triggers a **major** release.
- `chore:`, `docs:`, `ci:`, `test:`, `refactor:` -> Maintenance changes (no release bump).

Upon merging to `main`, the `release.yml` workflow automatically computes the next version, updates `version` and `appVersion` in `chart/Chart.yaml`, creates a Git tag, publishes a GitHub Release, and packages and pushes the Helm chart to GHCR.

## Support

If you find this project useful, consider supporting my work on [Ko-fi](https://ko-fi.com/joeckr):

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/joeckr)

## License

Please refer to the `LICENSE` file for details.
