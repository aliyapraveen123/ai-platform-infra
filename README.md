# AI Task Platform Infrastructure

Separate GitOps repository for Kubernetes manifests and Argo CD configuration.

## Layout

- `k8s/base/`: ConfigMap, MongoDB and Redis StatefulSets, app Deployments and Services, Ingress, and worker HPA
- `k8s/overlays/staging/`: staging namespace, hostname, replica count, and image tags
- `k8s/overlays/production/`: production namespace, hostname, replica count, and image tags
- `argocd/`: Argo CD Applications with automated sync, self-heal, and pruning

Application source lives in the sibling `venture_assignment` repository.

## Prerequisites

- The application and infrastructure GitHub repositories must be pushed to GitHub with the Actions workflows enabled
- A k3s cluster with metrics-server enabled (needed for CPU-based HPA)
- An ingress controller installed with class `nginx`
- Argo CD installed in the `argocd` namespace
- DNS records for `staging.tasks.example.com` and `tasks.example.com` pointing to the ingress controller
- Published images in Docker Hub

Replace `your-dockerhub-user` in each Kustomize image mapping with the Docker Hub namespace that CI publishes to. Replace `your-github-user` in both Argo CD Applications with the owner of this infrastructure repository. Replace the example ingress hosts with real DNS names and install a TLS certificate in each target namespace.

CI publishes immutable tags in the form `sha-<40-character-commit>`. The application repository's GitHub Actions workflow updates the three staging image tags in this repository; a separate manual workflow promotes the current staging tag to production. Avoid hand-editing image tags while CI is deploying.

The application repository must define `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `INFRA_REPOSITORY` (`owner/repo`), and `INFRA_REPO_TOKEN` (contents write access). Its production GitHub Actions environment should require reviewers. The infra repository's Argo CD applications read the changed Kustomize overlays and sync them automatically.

## Secrets

The JWT Secret is not stored in Git. Create it in each destination namespace before syncing the apps:

```bash
kubectl create namespace ai-task-platform-staging
kubectl create secret generic ai-task-platform-secrets \
	--namespace ai-task-platform-staging \
	--from-literal=JWT_SECRET="$(openssl rand -hex 32)"

kubectl create namespace ai-task-platform-production
kubectl create secret generic ai-task-platform-secrets \
	--namespace ai-task-platform-production \
	--from-literal=JWT_SECRET="$(openssl rand -hex 32)"
```

Keep production secrets in a managed secret store or sealed-secret workflow; do not commit plaintext credentials to Git. The database and Redis currently use cluster-internal networking without authentication and should be restricted to the application namespace/network policies in a hardened production cluster.

## Render and apply

Install `kubectl` with the k3s setup or use the `kubectl` bundled with the cluster. Kustomize is built into current kubectl releases:

```bash
kubectl kustomize k8s/overlays/staging
kubectl kustomize k8s/overlays/production
kubectl apply -f argocd/application-staging.yaml
kubectl apply -f argocd/application-production.yaml
```

Alternatively apply an overlay directly for a non-GitOps test:

```bash
kubectl apply -k k8s/overlays/staging
kubectl -n ai-task-platform-staging get pods,services,ingress,hpa
```

Do not run direct `kubectl apply` and Argo CD auto-sync as competing deployment owners after GitOps is enabled.

## Worker scaling

The worker Deployment starts with two replicas in the base, three in production, and is autoscaled by CPU utilization between three and twenty replicas in production. Scale workers independently when needed:

```bash
kubectl -n ai-task-platform-production get hpa worker
kubectl -n ai-task-platform-production scale deployment/worker --replicas=6
```

The HPA may later adjust manual replicas. Queue-depth-based scaling is preferable for bursty task workloads and can be added with KEDA once Redis monitoring metrics are available.

## Storage and operations

MongoDB and Redis use single-replica StatefulSets with persistent volume claims for k3s development. These are not highly available database deployments; production should use managed/replicated services, backups, restore drills, and network policies before critical workloads. The worker HPA needs cluster metrics and enough CPU/memory capacity on k3s nodes.