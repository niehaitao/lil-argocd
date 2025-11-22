# Lil ArgoCD

This repository defines the desired state of the Kubernetes platform using GitOps.

ArgoCD watches this repo and manages:

- whoami application (dev, staging, prod)
- cluster add-ons (Traefik, cert-manager, logging, monitoring)
- environment-specific overrides
- 
```bash
.
├── README.md
├── apps/
│   └── whoami/
│       ├── base/
│       │   ├── kustomization.yaml
│       │   └── values.yaml
│       └── overlays/
│           ├── dev/
│           │   ├── application.yaml
│           │   └── values.yaml
│           ├── staging/
│           │   ├── application.yaml
│           │   └── values.yaml
│           └── prod/
│               ├── application.yaml
│               └── values.yaml
└── infra/
    ├── traefik/
    ├── cert-manager/
    ├── monitoring/
    └── logging/
```

Designing a GitOps repository structure is a foundational decision that impacts security, scalability, and team autonomy. There isn't a single "best" way, but rather a set of **best practices and common patterns** tailored to your organization's size, complexity, and number of clusters/environments.

Here are the key considerations and popular structures.

-----

## 1. Monorepo vs. Multi-Repo (Polyrepo)

The first choice is whether to put all your Kubernetes configuration into a single repository or split it across multiple ones.

| Feature                | Monorepo (Single Repository)                                                               | Multi-Repo (Multiple Repositories)                                                                                      |
| :--------------------- | :----------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------- |
| **Simplicity**         | **High.** Easy to centralize, manage one set of permissions.                               | **Lower.** More repositories to manage, coordinate, and bootstrap.                                                      |
| **Ownership/Autonomy** | **Lower.** All teams/concerns are mixed. Access control is less granular.                  | **High.** Clear separation of concerns, allowing specific access controls (e.g., Platform Team Repo vs. App Team Repo). |
| **Performance**        | Can be **Slower** for GitOps tools (e.g., Argo CD) as the repository grows.                | **Faster.** Smaller repos mean quicker cloning and processing by the GitOps operator.                                   |
| **Blast Radius**       | **High.** A bad change can affect multiple clusters/environments.                          | **Lower.** Issues are isolated to one application or one concern.                                                       |
| **Best For**           | Small teams, few clusters, simple applications, or organizations starting out with GitOps. | Large organizations, many clusters/environments, strong separation of duties (Platform vs. Dev teams).                  |

-----

## 2. Recommended Directory Structures (for Monorepo)

For most organizations starting out, a well-structured Monorepo is the easiest way to begin. The best structures create clear separation by **Concern** (Platform vs. Application) and **Environment/Cluster**.

### Structure A: Separated by Concern (Platform vs. App)

This is a very common and highly recommended approach for clear separation of duties.

```bash
├── argocd/               # Argo CD specific configuration (Applications, AppProjects, ApplicationSets)
│   ├── infrastructure/   # Argo CD Application CRs for Cluster/Infra components
│   └── applications/     # Argo CD Application CRs for business apps
│
├── platform/             # Configuration for cluster-level tools (managed by Platform/Ops team)
│   ├── base/             # Base manifests (e.g., Cert-Manager, Ingress, Monitoring)
│   ├── dev-cluster/      # Kustomize overlay/Helm values for Dev cluster platform tools
│   └── prod-cluster/     # Kustomize overlay/Helm values for Prod cluster platform tools
│
└── apps/                 # Configuration for business applications (managed by Dev teams)
    ├── whoami/
    │   ├── base/         # Base Helm Chart/Kustomize Base
    │   ├── dev/          # Kustomize overlay/Helm values for Dev
    │   └── prod/         # Kustomize overlay/Helm values for Prod
    └── another-app/
        ├── base/
        └── staging/
```

**Key Takeaways:**

1.  **`argocd/`**: Contains the *declarative management* of Argo CD itself (`Application` objects). This is what Argo CD reads to know *what* to deploy.
2.  **`platform/`**: Contains the actual Kubernetes manifests for things like `cert-manager` or `monitoring`.
3.  **`apps/`**: Contains the actual Kubernetes manifests for your business applications.
4.  **DRY Principle**: Uses Kustomize or Helm (as shown with `base/`, `dev/`, `prod/`) to manage environment-specific differences without copying all the YAML.

### Structure B: Separated by Cluster/Environment (for multiple clusters)

This works well when you have distinct clusters for each environment (e.g., `staging-us-west`, `prod-eu-central`).

```bash
├── clusters/
│   ├── staging-cluster-1/
│   │   ├── system/       # Platform tools for this specific cluster (e.g., Cert-Manager, Secrets)
│   │   └── applications/ # Argo CD ApplicationSet/Application CRs pointing to app repos
│   └── prod-cluster-2/
│       ├── system/
│       └── applications/
│
└── applications/         # Centralized folder for all application manifests (often managed via Helm or Kustomize Base)
    ├── app-a/
    │   ├── base/
    │   └── overlays/
    │       ├── staging/
    │       └── prod/
    └── app-b/
        └── ...
```

**Key Takeaways:**

  * **Cluster-Centric View**: The top level (`clusters/`) gives you an immediate view of everything installed on a specific cluster.
  * **Promotion Flow**: Promotion happens by moving a Git reference (e.g., a specific Helm Chart version or a commit SHA) in the `applications/` section, and then updating the **overlays** referenced by the cluster definitions in `clusters/`.

-----

## 3. Universal GitOps Best Practices

Regardless of the structure you choose, these practices are essential:

  * **Separate Config from Source Code:** **NEVER** put your Kubernetes manifests in the same repository as your application's source code (Java/Python/Go, etc.). Keep a dedicated GitOps repository for configuration.
  * **Only One Main Branch:** Avoid using branches (e.g., `dev`, `staging`, `prod`) to represent environments. **Use directories** (e.g., `apps/whoami/dev`, `apps/whoami/prod`) and a single `main` branch. Environment promotion is done via **Pull Requests** that merge changes from a development path to a production path in the same branch.
  * **Never Store Plain Secrets:** Encrypt secrets (using tools like **Sealed Secrets** or **SOPS**) or manage them via an external system (like **Vault** or **External Secrets Operator**) where only a reference or the encrypted value is stored in Git.
  * **Use Declarative Configuration Tools (DRY):** Use **Kustomize** or **Helm** to avoid duplicating manifests across environments. Helm is better for reusability and third-party apps; Kustomize is better for simple layering/patching of raw YAML.
  * **Enforce Consistency:** Choose a structure and stick to it. Consistency is the most important factor for a maintainable GitOps setup.

Do you have multiple clusters, or are you primarily focused on managing different environments (dev/staging/prod) within a single cluster?