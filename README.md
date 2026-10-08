# Argo CD Nginx Demo 2

This project demonstrates **GitOps using Kubernetes, Argo CD, and GitHub**.

Argo CD continuously monitors the GitHub repository and automatically synchronizes Kubernetes resources with the desired state defined in Git.

## GitOps Flow

```text
Developer
    │
    │ git push
    ▼
GitHub Repository
    │
    │ Argo CD monitors
    ▼
Argo CD
    │
    │ Automated Sync
    ▼
Kubernetes Cluster
    │
    └── argocd-demo-2
         ├── Deployment
         └── Service
```

## 🛠️ Technologies

* Kubernetes
* Argo CD
* Git
* GitHub
* Nginx
* YAML
* kubectl

## 📁 Repository Structure

```text
argocd-nginx-demo-2/
│
├── deployment.yaml
├── service.yaml
├── application.yaml
└── README.md
```

---

# 1. Install Argo CD

### Create Argo CD namespace

```bash
kubectl create namespace argocd
```

### Install Argo CD

```bash
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Check Argo CD Pods

```bash
kubectl get pods -n argocd
```

### Check Argo CD Services

```bash
kubectl get svc -n argocd
```

### Check Argo CD Application CRD

```bash
kubectl get crd applications.argoproj.io
```

---

# 2. [Kubernetes Deployment](argocd-nginx-demo-2/deployment.yaml)

The `deployment.yaml` file creates an Nginx Deployment with 2 replicas.

```bash
kubectl apply -f deployment.yaml
```

Check the Deployment:

```bash
kubectl get deployment -n argocd-demo-2
```

---

# 3. [Kubernetes Service](argocd-nginx-demo-2/service.yaml)

The `service.yaml` file creates a ClusterIP Service for the Nginx Pods.

```bash
kubectl apply -f service.yaml
```

Check the Service:

```bash
kubectl get service -n argocd-demo-2
```

The Service selector:

```text
app: argocd-sample
```

matches the Pod label in the Deployment.

---

# 4. [Argo CD Application](argocd-nginx-demo-2/applicatin.yaml)

The `application.yaml` file defines the Argo CD Application.

```bash
kubectl apply -f application.yaml
```

### GitHub Repository

```yaml
repoURL: https://github.com/lok2003/argocd-nginx-demo-2.git
```

Argo CD uses this repository as the desired state.

### Branch

```yaml
targetRevision: main
```

Argo CD monitors the `main` branch.

### Path

```yaml
path: .
```

The Kubernetes manifests are located in the repository root.

### Destination Namespace

```yaml
namespace: argocd-demo-2
```

The Kubernetes resources are deployed into the `argocd-demo-2` namespace.

### Automated Sync

```yaml
automated:
```

Argo CD automatically synchronizes changes from Git.

### Prune

```yaml
prune: true
```

Resources removed from Git can also be removed from the Kubernetes cluster.

### Self-Healing

```yaml
selfHeal: true
```

If a managed Kubernetes resource is manually changed, Argo CD restores the Git-defined state.

### Create Namespace

```yaml
- CreateNamespace=true
```

Argo CD automatically creates `argocd-demo-2` if it does not exist.

---

# 5. Check Argo CD Application

```bash
kubectl get application -n argocd
```

Expected:

```text
NAME           SYNC STATUS   HEALTH STATUS
nginx-demo-2   Synced        Healthy
```

Check the specific Application:

```bash
kubectl get application nginx-demo-2 -n argocd
```

---

# 6. Check Kubernetes Resources

### All Resources

```bash
kubectl get all -n argocd-demo-2
```

### Pods

```bash
kubectl get pods -n argocd-demo-2
```

### Deployment

```bash
kubectl get deployment -n argocd-demo-2
```

### Service

```bash
kubectl get service -n argocd-demo-2
```

---

# 7. GitOps Workflow

The normal GitOps workflow is:

```text
Modify Kubernetes YAML
        ↓
git add
        ↓
git commit
        ↓
git push
        ↓
GitHub
        ↓
Argo CD detects change
        ↓
Argo CD synchronizes
        ↓
Kubernetes updated
```

### Git Commands

```bash
git status
```

```bash
git add .
```

```bash
git commit -m "update nginx configuration"
```

```bash
git push origin main
```

Check Argo CD:

```bash
kubectl get application nginx-demo-2 -n argocd
```

---

# 8. Self-Healing

The Application has:

```yaml
selfHeal: true
```

Git defines:

```yaml
replicas: 2
```

Manually change the Deployment:

```bash
kubectl scale deployment nginx-demo \
  --replicas=4 \
  -n argocd-demo-2
```

Argo CD detects the difference and restores the desired state:

```text
Git
replicas: 2
     ↓
Kubernetes
replicas: 4
     ↓
Argo CD detects drift
     ↓
Kubernetes
replicas: 2
```

---

# 9. Prune

The Application has:

```yaml
prune: true
```

If an Argo CD-managed resource is removed from Git, Argo CD can remove that resource from Kubernetes during synchronization.

Example:

```text
Git
 ├── Deployment
 └── Service
       ↓
Delete Service from Git
       ↓
git push
       ↓
Argo CD detects the change
       ↓
Service removed from Kubernetes
```

---

# 10. Troubleshooting

### Check Application Status

```bash
kubectl get application nginx-demo-2 -n argocd
```

### Describe Application

```bash
kubectl describe application nginx-demo-2 -n argocd
```

Check the `Events` section for synchronization errors.

### Check Argo CD Pods

```bash
kubectl get pods -n argocd
```

### Check Application Resources

```bash
kubectl get all -n argocd-demo-2
```

### Check Pod Details

```bash
kubectl describe pod <pod-name> -n argocd-demo-2
```

### Check Pod Logs

```bash
kubectl logs <pod-name> -n argocd-demo-2
```

---

# 11. Validate YAML Before Push

Validate the Deployment:

```bash
kubectl apply --dry-run=client -f deployment.yaml
```

Validate the Service:

```bash
kubectl apply --dry-run=client -f service.yaml
```

Validate the Argo CD Application:

```bash
kubectl apply --dry-run=client -f application.yaml
```

These commands validate the manifests without creating or updating resources.

---

# 12. Common Error Learned

During this project, an Argo CD synchronization failed because required Deployment fields were missing.

Argo CD reported errors such as:

```text
spec.selector: Required value
spec.template.metadata.labels: Invalid value
spec.template.spec.containers: Required value
```

The problem was identified using:

```bash
kubectl describe application nginx-demo-2 -n argocd
```

### Troubleshooting Flow

```text
Argo CD status
      ↓
kubectl describe application
      ↓
Read Events
      ↓
Find synchronization error
      ↓
Fix Git manifest
      ↓
git add .
      ↓
git commit
      ↓
git push
      ↓
Argo CD sync
```

---

# 13. Final Architecture

```text
                         GitHub
                            │
                            │
                 argocd-nginx-demo-2
                            │
                            ▼
                         Argo CD
                            │
                      nginx-demo-2
                            │
                            ▼
                    Kubernetes Cluster
                            │
                     argocd-demo-2
                            │
                  ┌─────────┴─────────┐
                  │                   │
              Deployment           Service
              nginx-demo        nginx-service
                  │
             ┌────┴────┐
             │         │
           nginx      nginx
            Pod        Pod
```

---

# 14. Useful kubectl Commands

### Argo CD Applications

```bash
kubectl get applications -n argocd
```

### Application Details

```bash
kubectl describe application nginx-demo-2 -n argocd
```

### Kubernetes Resources

```bash
kubectl get all -n argocd-demo-2
```

### Pods

```bash
kubectl get pods -n argocd-demo-2
```

### Deployment

```bash
kubectl get deployment -n argocd-demo-2
```

### Service

```bash
kubectl get service -n argocd-demo-2
```

### Namespaces

```bash
kubectl get namespaces
```

### Nodes

```bash
kubectl get nodes
```

### All Pods

```bash
kubectl get pods -A
```

---

# 15. Git Commands

### Check Status

```bash
git status
```

### Check Commit History

```bash
git log --oneline
```

### Push Changes

```bash
git add .
git commit -m "update manifests"
git push origin main
```

---

# 16. Learning Outcomes

This project demonstrates practical knowledge of:

* Kubernetes Deployments
* Kubernetes Services
* Kubernetes namespaces
* YAML manifests
* Git and GitHub
* Argo CD Applications
* GitOps
* Automated synchronization
* Self-healing
* Resource pruning
* Argo CD troubleshooting
* Git as the source of truth
* Kubernetes desired state vs actual state

---

## 🎯 Key DevOps Concept

**Git is the source of truth.**

Argo CD continuously compares:

```text
Git Desired State
        VS
Kubernetes Actual State
```

If they are different, Argo CD reconciles the Kubernetes cluster toward the desired state defined in Git.

This project demonstrates the fundamental **GitOps workflow**:

```text
GitHub → Argo CD → Kubernetes
```
