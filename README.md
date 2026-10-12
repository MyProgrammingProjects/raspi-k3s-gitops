# Argo CD GitOps Repository

## Overview

This repository contains the GitOps configuration used to manage application deployments within my Raspberry Pi Kubernetes cluster.

The objective is to move from imperative deployment workflows toward a declarative GitOps model where the desired cluster state is defined in Git and continuously reconciled by Argo CD.

The platform currently consists of:

* K3s Kubernetes cluster
* Argo CD (Core installation)
* Helm-based application deployments
* Azure Container Registry (ACR)
* Cloudflare Tunnel for external access

---

# Architecture

Argo CD continuously monitors application repositories and reconciles the cluster state whenever changes are detected.

```text
Git Repository
      │
      ▼
 Argo CD
      │
      ▼
 Kubernetes API
      │
      ▼
 Deployments
 Services
 Secrets
 ConfigMaps
 StatefulSets
```

Applications remain independently versioned in dedicated repositories while Argo CD acts as the deployment orchestrator.

---

# Why Argo CD Core

This environment uses the Argo CD Core installation rather than the full platform.

Reasons:

* Reduced resource consumption
* Better suitability for Raspberry Pi hardware
* CLI-driven operation
* Focus on GitOps reconciliation rather than UI features

Installed components include:

* Application Controller
* Repository Server
* ApplicationSet Controller
* Redis

The Argo CD web UI and SSO components were intentionally omitted.

---

# Repository Structure

Each application is managed through a dedicated Argo CD Application manifest.

```text
projects/
├── microservices-db-project-argocd.yaml
└── microservices-project-argocd.yaml

applications/
├── postgres-argocd.yaml
├── gateway-argocd.yaml
├── accounts-argocd.yaml
├── users-argocd.yaml
└── search-argocd.yaml
```

This approach keeps manifests:

* small
* isolated
* easier to maintain
* independently deployable


### Argo CD Projects and Deployment Guardrails

Previously, all applications relied on Argo CD's `default` project. I introduced two dedicated AppProjects to establish explicit deployment boundaries for the microservices and their PostgreSQL database.

#### Source Repositories

Both projects restrict their applications to the following Git repository:

- `git@github.com:MyProgrammingProjects/raspi-k3s-helm-charts.git`

This limits the Git repository from which applications managed by these projects can retrieve their deployment configuration.

#### Deployment Destinations

Each project is restricted to a specific namespace on the local Kubernetes cluster:

- **microservices-project:** `prod`
- **microservices-db-project:** `databases`

Both destinations use `https://kubernetes.default.svc` as the cluster identifier.

This separates the deployment targets for the application workloads and the database.

#### Kubernetes Resource Restrictions

The projects explicitly allow the following namespaced resource kinds:

- **microservices-project:** Deployments, Services, ConfigMaps, and Secrets.
- **microservices-db-project:** StatefulSets, Services, ConfigMaps, and Secrets.

Both projects allow the creation of the `Namespace` cluster-scoped resource. They also explicitly blacklist PersistentVolumes, StorageClasses, ClusterRoles, and ClusterRoleBindings.

These restrictions limit the types of Kubernetes resources that applications can manage through Argo CD. They also prevent these projects from managing the explicitly blacklisted resource kinds.

#### Project-Scoped RBAC (Design)

The AppProject manifests also include example role definitions to illustrate how project-scoped RBAC can be structured.

The intended configuration defines two roles:

- **dev-readonly:** Intended to allow the `microservices-devs` group to view applications within the project.
- **platform-admin:** Intended to allow the `platform-team` group to perform all application actions within the project.

These definitions demonstrate how Argo CD application permissions can be scoped to individual projects, separating GitOps operations from Kubernetes RBAC.

**Limitation:** This RBAC configuration is illustrative, not an active access-control mechanism in my current Argo CD Core installation. The role definitions document the intended authorization model, but they should not be interpreted as enforced permissions in this environment. The reason is that Argo CD RBAC model is a feature not available on Argo CD Core.

Together, these AppProjects establish explicit boundaries for repository access, deployment destinations, permitted resource kinds, and Argo CD application operations.

These projects allow me to restrict the repositories applications can use, their deployment destinations, and the Kubernetes resource kinds they are permitted to manage.

Separating the project definitions from the application manifests also makes these policies easier to review and maintain alongside the rest of the GitOps configuration.

The next step is to use project-scoped RBAC roles to define which authenticated identities or automation clients can view and operate applications within each project. This is GitOps RBAC, not Kubernetes RBAC. 


Argo CD manifest for projects and applications can be found on the [argocd](https://github.com/MyProgrammingProjects/raspi-k3s-gitops/tree/main/argocd) folder

---

# GitOps Workflow

The deployment process follows the workflow below:

```text
Developer Push
      │
      ▼
 Application Repository
      │
      ▼
 Argo CD Detects Change
      │
      ▼
 Manifest Generation
      │
      ▼
 Cluster Reconciliation
      │
      ▼
 Application Updated
```

Automatic synchronization is enabled for all applications.

Features used:

* Automated Sync
* Self-Healing
* Automatic Pruning
* Namespace Creation

Example:

```yaml
syncPolicy:
  automated:
    selfHeal: true
    prune: true
  syncOptions:
    - CreateNamespace=true
```

---

# Private Repository Access

Application repositories are private.

Argo CD authenticates using SSH keys registered against GitHub repositories.

Repositories are registered using:

```bash
argocd repo add git@github.com:<repository>.git \
  --ssh-private-key-path ~/.ssh/argocd_id_ed25519
```

This allows Argo CD to securely retrieve deployment manifests and Helm charts.

---

# Managed Applications

Current applications managed by Argo CD:

| Application  | Namespace | Type        |
| ------------ | --------- | ----------- |
| PostgreSQL   | databases | StatefulSet |
| Gateway API  | prod      | Deployment  |
| Accounts API | prod      | Deployment  |
| Users API    | prod      | Deployment  |
| Search API   | prod      | Deployment  |

Each application is sourced directly from its own Git repository.

---

# Container Registry Integration

Container images are stored in Azure Container Registry (ACR).

The registry is private and Kubernetes authenticates through:

* ACR Token
* Kubernetes Docker Registry Secret
* Dedicated Service Account

This allows Argo CD deployments to pull private container images without exposing credentials inside application manifests.

---

# Validation

The deployment was validated through the following tests.

## Argo CD Synchronization

All applications successfully reached:

```text
Status: Synced
Health: Healthy
```

## Database Deployment

Validation included:

* PostgreSQL StatefulSet creation
* Service creation
* Secret management
* Persistent volume usage

## Persistent Storage Verification

A test table was created inside PostgreSQL.

The database pod was then deleted and recreated.

After restart:

* data remained available
* table persisted correctly

This verified proper persistent volume functionality.

## Service-to-Service Communication

Cluster-internal communication was validated using temporary test containers.

Successful tests included:

* User registration
* Authentication
* JWT generation
* User search operations

## Gateway Routing

Requests were validated both:

* Directly against internal services
* Through the Gateway API

This confirmed correct service discovery and reverse proxy routing.

---

# Lessons Learned

Several implementation issues were encountered during deployment:

* Repository URL mismatches (HTTPS vs SSH)
* Incorrect Git branch configuration
* Helm chart path errors
* Namespace inconsistencies
* Private registry authentication failures
* Shared Secret and ConfigMap naming collisions

These issues reinforced the importance of:

* deterministic resource naming
* environment isolation
* repository standardization
* declarative configuration management

---

# Current State

The cluster is currently managed through a GitOps workflow where Argo CD continuously reconciles:

* PostgreSQL database deployment
* Gateway API
* Accounts API
* Users API
* Search API

All applications are deployed from Git repositories and maintained in a synchronized state through automated reconciliation.
Applied manifests to create the two new Argo CD projects

![Argo CD](./images/argocd-projects-manifest-applied.png "argocd configurations")


Argo CD application now present projects that were created and not _default_  project.

![Argo CD](./images/argocd-applications.png "argocd configurations")
