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
- **Compose (`compose.yml`)**:
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
├── compose.yml                  # Local lab service definition
├── hk.pkl                       # hk git hooks and checks configuration
├── mise.toml                    # Mise tools and tasks
└── README.md
```

---

## Security & Compliance Architecture

Both OpenShift and Talos Linux prioritize workload security and least privilege, but they enforce and evaluate constraints through different mechanisms. This repository is architected to satisfy both environments without configuration changes.

### OpenShift Compliance (`restricted-v2` SCC)

OpenShift uses **Security Context Constraints (SCC)** to control pod permissions. Under the default `restricted-v2` SCC:
- **Arbitrary Dynamic UIDs**: OpenShift assigns a random UID from a dedicated per-namespace range (e.g., `1000670000`). Containers cannot assume a fixed UID like `1000`.
- **Root Group (GID 0)**: Files and directories required at runtime must be accessible by group 0 (`root` group) with group read/write permissions (`g+rwX`).
- **Dropped Capabilities**: Drops standard root capabilities (`CHOWN`, `DAC_OVERRIDE`, `FOWNER`, `SETUID`, `SETGID`, `SYS_CHROOT`, etc.) and permits only unprivileged operations (and `NET_BIND_SERVICE` when needed).
- **Unprivileged Ports**: Containers must listen on non-privileged ports (> 1024), such as port `8080`.
- **Routing**: OpenShift natively supports `route.openshift.io/v1` Routes for external traffic via `ingress.route: "true"`.

### Talos Linux Compliance (Kubernetes PSS `restricted`)

Talos Linux is an immutable, minimal, secure-by-default Kubernetes operating system with no SSH, no interactive shell, and an immutable root filesystem. In Talos clusters:
- **Pod Security Standards (PSS)**: Workload namespaces enforce the Kubernetes **Pod Security Admission (PSA)** `restricted` profile.
- **Must Run As Non-Root**: The pod specification sets `securityContext.runAsNonRoot: true`. Containers cannot execute as UID 0.
- **Drop All Capabilities**: The container specification explicitly drops all Linux capabilities (`capabilities: drop: ["ALL"]`).
- **Disallow Privilege Escalation**: Sets `securityContext.allowPrivilegeEscalation: false` to prevent child processes from acquiring more privileges than the parent.
- **Seccomp Profile**: Pods enforce `seccompProfile: { type: RuntimeDefault }`.
- **Credential Protection**: Hardened with `automountServiceAccountToken: false` to avoid leaking Kubernetes API tokens to application containers.
- **Standard Ingress & Routing**: Talos relies on standard Kubernetes `networking.k8s.io/v1` `Ingress` (configured when `ingress.route: "false"`).

### Compliance Matrix

| Security Dimension | OpenShift (`restricted-v2` SCC) | Talos Linux (Kubernetes PSS `restricted`) | Implementation in This Repo |
|---|---|---|---|
| **Workload Execution** | Unprivileged non-root | Non-root UID (`runAsNonRoot: true`) | Unprivileged container image (`nginx-unprivileged`) + `runAsNonRoot: true` |
| **Group Permissions** | Requires GID 0 (`root`) with `g+rwX` | Compatible with GID 0 / unprivileged groups | Configured for arbitrary UIDs and group 0 compatibility |
| **Capabilities** | Drops root caps; allows `NET_BIND_SERVICE` | Must drop `ALL` capabilities | `capabilities.drop: ["ALL"]` in Helm chart |
| **Privilege Escalation** | Prohibited | `allowPrivilegeEscalation: false` | Configured in Helm `securityContext` |
| **Seccomp Profile** | `RuntimeDefault` | `RuntimeDefault` or `Localhost` | `seccompProfile: { type: RuntimeDefault }` |
| **Service Account Token** | Optional | Recommended disabled | `automountServiceAccountToken: false` in pod spec |
| **Port Binding** | Unprivileged (> 1024) | Unprivileged (> 1024) | Listens on port `8080` |
| **Ingress Layer** | OpenShift Route (`route.openshift.io/v1`) | Kubernetes Ingress (`networking.k8s.io/v1`) | Configurable via `ingress.route: "true"` or `"false"` |

---

## Local Environment & Podman Setup

To ensure containerized applications and Helm charts tested locally run cleanly when deployed to OpenShift or Talos Linux, this repository is designed to be used alongside the Podman configuration in [joeckr/dotfiles](https://github.com/joeckr/dotfiles).

The dotfiles repository provides a centralized [`containers.conf`](https://github.com/joeckr/dotfiles/blob/main/containers/containers.conf) (deployed to `~/.config/containers/containers.conf`) that configures Podman to simulate OpenShift and Talos Linux runtime restrictions:

| Security Rule | Podman Configuration | Description |
|---|---|---|
| **Random UID (`MustRunAsRange`)** | `userns = "auto"` | Allocates dynamic subordinate UID/GID ranges from `/etc/subuid` and `/etc/subgid`. Containers run unprivileged without mapping host root. |
| **Drop Capabilities** | `default_capabilities = ["NET_BIND_SERVICE"]` | Drops standard root capabilities (`CHOWN`, `DAC_OVERRIDE`, `FOWNER`, `SETUID`, `SETGID`, `SYS_CHROOT`, etc.) and permits only `NET_BIND_SERVICE`. |
| **Disallow Privileged** | `privileged = false` | Disallows privileged container execution by default. |
| **Seccomp Profile** | `seccomp_profile = "/usr/share/containers/seccomp.json"` | Enforces the runtime default seccomp profile (`RuntimeDefault`). |
| **Namespace Isolation** | `cgroupns`, `ipcns`, `pidns`, `utsns = "private"` | Enforces private container namespaces (host namespaces are forbidden in restricted profiles). |

### macOS Podman Machine Integration

On macOS, the dotfiles installer script (`brew/podman.sh`) automates the machine lifecycle:

1. Deploys `containers/containers.conf` to `~/.config/containers/containers.conf` on the host.
2. Initializing `podman machine init` automatically mounts `~/.config/containers` into `/etc/containers` inside the Fedora CoreOS VM.
3. Automatically symlinks `/etc/containers/containers.conf` to the VM user's config (`~core/.config/containers/containers.conf`) and restarts the Podman API service so all container executions immediately enforce these constraints.

---

## Testing & Validation Process

This repository defines a 3-tier testing process to validate container security, manifest generation, and runtime compatibility from local development through to production cluster deployment.

```
┌─────────────────────────┐     ┌─────────────────────────┐     ┌─────────────────────────┐
│ Tier 1: Local Container │ ──> │ Tier 2: Podman Play     │ ──> │ Tier 3: Talos Cluster   │
│ Fast local prototyping  │     │ Validate K8s manifests  │     │ Live Helm verification  │
│ (compose.yml)           │     │ (podman play kube)      │     │ (helm install)          │
└─────────────────────────┘     └─────────────────────────┘     └─────────────────────────┘
```

### Tier 1: Local Container Validation (`compose.yml`)

The [`compose.yml`](compose.yml) configuration runs the service container locally using Podman Compose:

```sh
# Start container in detached mode
mise run compose
# or: podman compose up -d

# View container logs
mise run logs
# or: podman compose logs -f

# Verify connectivity
curl http://localhost:8080

# Stop container
mise run down
# or: podman compose down
```

**What this verifies:**
- Unprivileged user execution without host root privileges.
- Unprivileged port binding on port `8080`.
- Health check verification and local volume mount handling.

---

### Tier 2: Local Kubernetes Manifest Testing (`mise run play`)

Before deploying to an actual Kubernetes cluster, you can test the rendered Kubernetes manifests locally using Podman's built-in `play kube` feature.

```sh
# Render templates and play Kubernetes manifests locally
mise run play

# Teardown the played pod and resources
mise run downplay
```

**How `mise run play` works:**
1. Triggers the dependent task `mise run helm-template`, which executes:
   ```sh
   helm dependency build chart/
   helm template test chart/ > rendered.yaml
   ```
2. Executes `podman play kube rendered.yaml`, which:
   - Reads the multi-document Kubernetes YAML (`Deployment`, `Service`).
   - Creates a local Podman pod matching the Kubernetes `Deployment` specification.
   - Applies the pod's `securityContext` (`runAsNonRoot: true`, capabilities drop, seccomp profile).
   - Exposes container port `8080`.

**Inspecting the local play deployment:**
```sh
# View running pods created by play kube
podman pod ps

# View container status within the pod
podman ps --filter "pod=lab-service"

# Verify service response
curl http://localhost:8080

# Check container logs within the pod
podman logs -f lab-service-pod-lab-service
```

**Teardown:**
```sh
mise run downplay
# or: podman play kube rendered.yaml --down
```

---

### Tier 3: Cluster Deployment & Testing on Talos Linux (`mise run helm-install`)

The final phase validates the workload on a live **Talos Linux** Kubernetes cluster. This tests real-world Pod Security Admission (PSA) enforcement, network routing, and container startup under production constraints.

#### 1. Cluster Prerequisites & Configuration

Ensure your `kubectl` context points to your Talos cluster:
```sh
kubectl config current-context
# Example: admin@my-talos-cluster
```

Verify that the target namespace enforces the `restricted` Pod Security Standard:
```sh
kubectl get ns default --show-labels
# Should include:
# pod-security.kubernetes.io/enforce=restricted
# pod-security.kubernetes.io/enforce-version=latest
```

#### 2. Linting & Template Validation

Run the linter and inspect the generated manifests before cluster deployment:
```sh
mise run helm-lint
mise run helm-template

# Test OpenShift Route rendering
helm template test chart/ --set ingress.enabled=true --set ingress.route="true"
```

#### 3. Deploying to the Talos Cluster

Install the Helm chart release:
```sh
mise run helm-install
# or: helm install test chart/
```

#### 4. Verifying Talos PSS Compliance & Health

Check the pod status and verify that Talos Linux Pod Security Admission (PSA) allowed the pod to run:

```sh
# Check pod deployment status
kubectl get pods -l app=lab-service

# Inspect pod details and events for security policy rejections
kubectl describe pod -l app=lab-service
```

> [!TIP]
> If your namespace enforces the `restricted` Pod Security Standard and there are non-compliant settings (such as missing `runAsNonRoot` or un-dropped capabilities), `kubectl describe pod` will show warning events from the `pod-security` admission controller.

Check the application logs:
```sh
kubectl logs -l app=lab-service -f
```

Verify service access via port-forwarding:
```sh
# Forward port 8080 from the cluster pod
kubectl port-forward svc/lab-service 8080:8080

# In a separate shell, test HTTP response
curl http://localhost:8080
```

#### 5. Uninstalling from the Talos Cluster

When testing is complete, clean up the release:
```sh
mise run helm-uninstall
# or: helm uninstall test
```

---

## Local Development & Tasks

### Setup

```bash
# Install tools and git hooks
mise run install
```

### Available Tasks

Run tasks with `mise run <task>`:

| Task | Description | Command |
|---|---|---|
| `install` | Install tools and set up git hooks | `hk install --mise` |
| `hk` (or `check`) | Run all linters and hook checks | `hk check --all` |
| `compose` | Start local container stack with Podman Compose | `podman compose up -d` |
| `down` | Stop local Podman Compose stack | `podman compose down` |
| `logs` | View Podman Compose logs | `podman compose logs -f` |
| `play` | Test Helm chart manifests locally with Podman Play Kube | `podman play kube rendered.yaml` |
| `downplay` | Stop and remove Podman Play Kube pods | `podman play kube rendered.yaml --down` |
| `helm-dep` | Build Helm chart dependencies | `helm dependency build chart/` |
| `helm-lint` | Lint Helm chart | `helm lint chart/` |
| `helm-template` | Render Helm chart templates to `rendered.yaml` | `helm template test chart/ > rendered.yaml` |
| `helm-install` | Install Helm chart to current Kubernetes cluster | `helm install test chart/` |
| `helm-uninstall` | Uninstall Helm chart release from cluster | `helm uninstall test` |
| `trivy-fs` | Scan repository filesystem for security vulnerabilities | `trivy fs .` |

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
