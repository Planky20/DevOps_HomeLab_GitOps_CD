# DevOps Home Lab — GitOps CD

This repository contains the Kubernetes application manifests for my AKS DevOps home lab.

The goal of this repository is to act as the GitOps source of truth for the application deployed to Azure Kubernetes Service. Argo CD watches this repository and continuously reconciles the desired Kubernetes state with the actual state running in the cluster.

---

## Table of Contents

* [Project Context](#project-context)
* [Quick Start](#quick-start)
* [What This Repository Provides](#what-this-repository-provides)
* [Repository Structure](#repository-structure)
* [GitOps Role](#gitops-role)
* [Kubernetes Manifests](#kubernetes-manifests)

  * [Deployment](#deployment)
  * [Service](#service)
  * [Ingress](#ingress)
* [Application Hardening](#application-hardening)
* [Image Update Flow](#image-update-flow)
* [Argo CD Behavior](#argo-cd-behavior)
* [Ingress Access](#ingress-access)
* [Validation Commands](#validation-commands)
* [Restore Flow Notes](#restore-flow-notes)
* [Security Notes](#security-notes)
* [Related Repositories](#related-repositories)
* [Current Status](#current-status)

---

## Project Context

This repository is one part of a multi-repository AKS DevOps home lab:

```text
DevOps_HomeLab_Terraform
  → Azure infrastructure provisioning and platform add-on configuration

DevOps_HomeLab_CI
  → Application source code, Docker image build, ACR push and GitOps repository update

DevOps_HomeLab_GitOps_CD
  → Kubernetes desired state managed by Argo CD

DevOps_HomeLab_Documentation
  → Main project overview, architecture diagram and implementation phase notes
```

The full deployment flow is:

```text
Azure DevOps CI
  → builds a Docker image
  → pushes it to Azure Container Registry
  → updates the image tag in this GitOps repository

Argo CD
  → watches this repository
  → detects the Git change
  → synchronizes the desired state to AKS

AKS
  → runs the updated application workload
```

> [!NOTE]
> This repository does not contain application source code or infrastructure code. It only contains the Kubernetes desired state for the application.

---

## Quick Start

This repository is not normally applied manually with `kubectl apply`.

The intended flow is:

```text
1. Azure DevOps pipeline updates deployment.yaml with a new image tag
2. The change is committed to this repository
3. Argo CD detects the commit
4. Argo CD synchronizes the application to AKS
```

To verify the application after Argo CD sync:

```powershell
kubectl get application aks-lab-app -n argocd
kubectl get deployments
kubectl get pods
kubectl get svc
kubectl get ingress
```

To check which image tag is currently deployed:

```powershell
kubectl get deployment aks-lab-app `
  -o jsonpath="{.spec.template.spec.containers[0].image}"
```

> [!IMPORTANT]
> The Git repository is the source of truth. Manual changes made directly in the cluster can be reverted by Argo CD self-heal.

---

## What This Repository Provides

This repository defines:

* Kubernetes Deployment for the application
* Kubernetes ClusterIP Service
* Kubernetes Ingress resource
* resource requests and limits
* readiness and liveness probes
* rolling update strategy
* desired container image reference used by Argo CD

---

## Repository Structure

```text
.
├── aks-lab-app/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
│
└── README.md
```

---

## GitOps Role

This repository represents the desired state of the application.

Instead of deploying directly to AKS from the CI pipeline, the pipeline updates this repository. Argo CD then applies the Kubernetes manifests to the cluster.

This creates a clean separation between CI and CD:

| Area              | Responsibility                                           |
| ----------------- | -------------------------------------------------------- |
| Azure DevOps CI   | Build image, push image to ACR, update GitOps repository |
| GitOps repository | Store desired Kubernetes application state               |
| Argo CD           | Reconcile Git state with AKS state                       |
| AKS               | Run the application workload                             |

This approach makes application changes auditable because every deployment is represented by a Git commit.

---

## Kubernetes Manifests

The application is defined in the `aks-lab-app` directory.

### Deployment

The Deployment manifest is located at:

```text
aks-lab-app/deployment.yaml
```

It defines:

* application container image
* replica count
* rolling update strategy
* revision history limit
* container port
* CPU and memory requests
* CPU and memory limits
* readiness probe
* liveness probe

The image reference is automatically updated by the Azure DevOps CI pipeline.

Example image format:

```text
akslabwl.azurecr.io/aks-lab-app:<build-id>
```

### Service

The Service manifest is located at:

```text
aks-lab-app/service.yaml
```

The application is exposed inside the cluster using a `ClusterIP` service.

This is intentional because external traffic is handled by the NGINX Ingress Controller rather than by exposing the application directly with a `LoadBalancer` service.

### Ingress

The Ingress manifest is located at:

```text
aks-lab-app/ingress.yaml
```

It routes HTTP traffic to the application through the NGINX Ingress Controller.

The application uses the host:

```text
aks-lab.local
```

The Ingress Controller itself is installed as a platform add-on from the Terraform repository.

---

## Application Hardening

The Deployment includes several workload hardening settings.

### Resource Requests and Limits

The application container defines CPU and memory requests and limits.

This helps Kubernetes schedule the pod more predictably and prevents the container from using unlimited resources.

### Readiness Probe

The readiness probe checks whether the application is ready to receive traffic.

If the readiness probe fails, Kubernetes removes the pod from Service endpoints until it becomes ready again.

### Liveness Probe

The liveness probe checks whether the application process is still healthy.

If the liveness probe fails, Kubernetes restarts the container.

### Rolling Update Strategy

The Deployment uses a rolling update strategy to make application updates safer.

The configuration allows Kubernetes to create a replacement pod before terminating the old one, which reduces the risk of downtime during application updates.

### Revision History Limit

The Deployment keeps a limited number of previous ReplicaSet revisions.

This avoids unnecessary accumulation of old ReplicaSets while still keeping rollback history available.

> [!NOTE]
> This phase was implemented to make the workload more production-like without adding unnecessary components that the application does not currently need.

---

## Image Update Flow

The Azure DevOps CI pipeline updates the image tag in:

```text
aks-lab-app/deployment.yaml
```

The flow is:

```text
1. Application source code changes in the CI repository
2. Azure DevOps builds a new Docker image
3. The image is pushed to Azure Container Registry
4. The pipeline updates deployment.yaml in this repository
5. The pipeline commits and pushes the new image tag
6. Argo CD detects the Git change
7. Argo CD synchronizes the updated Deployment to AKS
```

This means the CI pipeline does not deploy directly to Kubernetes.

Instead, it updates Git, and Argo CD performs the deployment based on the desired state stored in this repository.

---

## Argo CD Behavior

The Argo CD Application is defined in the Terraform repository:

```text
DevOps_HomeLab_Terraform/argocd/applications/aks-lab-app.yaml
```

It points to this repository and watches the following path:

```text
aks-lab-app
```

Argo CD is configured with:

* automated synchronization
* self-heal enabled
* prune disabled for safety

### Automated Synchronization

When a new image tag is committed to this repository, Argo CD automatically applies the change to AKS.

### Self-Heal

If a Kubernetes resource is manually changed in the cluster, Argo CD can detect the drift and restore the desired state from Git.

### Prune Disabled

Prune is intentionally disabled for safety in this lab.

This means Argo CD will not automatically delete cluster resources that disappear from Git unless pruning is explicitly enabled later.

---

## Ingress Access

The application is exposed through NGINX Ingress using the host:

```text
aks-lab.local
```

To test the application without modifying the local hosts file, use:

```powershell
curl.exe -H "Host: aks-lab.local" http://<ingress-external-ip>/
```

To access it in a browser, add a local hosts file entry:

```text
<ingress-external-ip> aks-lab.local
```

On Windows, the hosts file is located at:

```text
C:\Windows\System32\drivers\etc\hosts
```

Then open:

```text
http://aks-lab.local
```

> [!NOTE]
> The external IP can change after the environment is destroyed and recreated.

---

## Validation Commands

Check Argo CD Application status:

```powershell
kubectl get application aks-lab-app -n argocd
```

Check application resources:

```powershell
kubectl get deployments
kubectl get pods
kubectl get svc
kubectl get ingress
```

Check the deployed image:

```powershell
kubectl get deployment aks-lab-app `
  -o jsonpath="{.spec.template.spec.containers[0].image}"
```

Check rollout status:

```powershell
kubectl rollout status deployment/aks-lab-app
```

Describe the pod to verify probes and resource limits:

```powershell
kubectl describe pod <pod-name>
```

Expected workload configuration includes:

```text
Requests:
  cpu:     50m
  memory: 64Mi

Limits:
  cpu:     200m
  memory: 128Mi
```

---

## Restore Flow Notes

During a full environment recreate, the Azure Container Registry is destroyed and recreated by Terraform.

Because of that, old image tags are not preserved in ACR.

The restore sequence is:

```text
1. Terraform recreates AKS and ACR
2. Azure DevOps pipeline rebuilds and pushes a fresh image to ACR
3. Azure DevOps updates deployment.yaml in this repository with the new image tag
4. Platform add-ons are installed
5. Argo CD reads this public GitOps repository
6. Argo CD deploys the application to AKS
```

> [!IMPORTANT]
> After ACR is recreated, the CI pipeline must run before Argo CD can successfully deploy the application. The Deployment must reference an image tag that exists in the recreated registry.

This repository is public so Argo CD can read it without storing GitHub credentials inside the cluster.

If the repository were private, Argo CD repository credentials would need to be configured after reinstalling Argo CD.

---

## Security Notes

This repository does not contain:

* secrets
* GitHub tokens
* Azure DevOps PATs
* kubeconfig files
* Terraform state
* container registry credentials

The manifests only contain Kubernetes desired state and public resource references required for the lab.

Sensitive values should not be stored in this repository.

---

## Related Repositories

This repository is part of a larger AKS DevOps home lab.

| Repository                                                                               | Responsibility                                                                   |
| ---------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| [DevOps_HomeLab_Terraform](https://github.com/Planky20/DevOps_HomeLab_Terraform)         | Azure infrastructure and platform add-ons                                        |
| [DevOps_HomeLab_CI](https://github.com/Planky20/DevOps_HomeLab_CI)                       | Docker build, ACR push and GitOps repository update                              |
| [DevOps_HomeLab_GitOps_CD](https://github.com/Planky20/DevOps_HomeLab_GitOps_CD)         | Kubernetes desired state for Argo CD                                             |
| [DevOps_HomeLab_Documentation](https://github.com/Planky20/DevOps_HomeLab_Documentation) | Main project documentation, architecture overview and implementation phase notes |

---

## Current Status

Implemented and validated:

* Kubernetes Deployment manifest
* ClusterIP Service manifest
* Ingress manifest for NGINX Ingress Controller
* resource requests and limits
* readiness and liveness probes
* rolling update strategy
* Argo CD automated synchronization
* Argo CD self-heal behavior
* GitOps image tag update from Azure DevOps CI
* application access through NGINX Ingress
* successful deployment after full infrastructure recreate
