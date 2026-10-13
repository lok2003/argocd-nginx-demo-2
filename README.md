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

# 2. [Kubernetes Deployment](/deployment.yaml)

The `deployment.yaml` file creates an Nginx Deployment with 2 replicas.

```bash
kubectl apply -f deployment.yaml
```

Check the Deployment:

```bash
kubectl get deployment -n argocd-demo-2
```

---

# 3. [Kubernetes Service](/service.yaml)

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

# 4. [Argo CD Application](/application.yaml)

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
---

# 16. Deployment Rollout Monitoring

Kubernetes provides commands to monitor Deployment rollouts and verify whether application updates have completed successfully.

### Check Rollout Status

```bash
kubectl rollout status deployment/httpd-demo -n argocd-demo-2
```

Expected output:

```text
deployment "httpd-demo" successfully rolled out
```

This confirms that the Deployment rollout has completed successfully.

### Check Deployment Revision

```bash
kubectl get deployment httpd-demo -n argocd-demo-2 -o jsonpath='{.metadata.annotations.deployment\.kubernetes\.io/revision}'
```

This command displays the current Deployment revision number.

### Check Rollout History

```bash
kubectl rollout history deployment/httpd-demo -n argocd-demo-2
```

Example output:

```text
deployment.apps/httpd-demo
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
```

Kubernetes stores Deployment rollout revisions when the Pod template changes.

### Inspect a Specific Revision

```bash
kubectl rollout history deployment/httpd-demo --revision=7 -n argocd-demo-2
```

This displays the saved Pod template for the specified revision, including the container image and other recorded Pod configuration.

### Inspect ReplicaSets

```bash
kubectl get replicasets -n argocd-demo-2
```

To inspect a specific ReplicaSet:

```bash
kubectl get replicaset <replicaset-name> -n argocd-demo-2 -o yaml
```

The ReplicaSet YAML contains the Pod template, including labels, containers, images, ports, volumes, mounts, node selectors, and tolerations when configured.

**Important:** Old revisions may no longer be available because Kubernetes retains only a limited amount of rollout history by default. Revision numbers can also skip numbers.

---

# 17. Deployment Rollback Using kubectl

Kubernetes supports rolling back a Deployment to a previously recorded revision.

### Roll Back to a Specific Revision

```bash
kubectl rollout undo deployment/httpd-demo --to-revision=7 -n argocd-demo-2
```

This command attempts to restore the Pod template recorded in revision 7.

### Roll Back to the Previous Revision

```bash
kubectl rollout undo deployment/httpd-demo -n argocd-demo-2
```

This command rolls back to the previous available revision.

### Verify the Rollback

Check the current image:

```bash
kubectl get deployment httpd-demo -n argocd-demo-2 -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Check rollout status:

```bash
kubectl rollout status deployment/httpd-demo -n argocd-demo-2
```

Check the Pods:

```bash
kubectl get pods -n argocd-demo-2
```

Check the revision history:

```bash
kubectl rollout history deployment/httpd-demo -n argocd-demo-2
```

### Kubernetes Rollback vs. GitOps Rollback

**Kubernetes rollback:**

```bash
kubectl rollout undo deployment/httpd-demo -n argocd-demo-2
```

Changes the live Deployment to a previously recorded revision. It does not change the Git repository.

**GitOps rollback:**

```bash
git log --oneline
git revert <commit-id>
git push origin main
```

Creates a new Git commit that reverses the selected commit. Argo CD can then synchronize the desired state from Git.

**Important:** This project uses Argo CD automated synchronization and self-healing. A manual Kubernetes rollback can be overwritten when Argo CD reconciles the Deployment with Git. For a GitOps-managed application, normally perform the rollback through Git.

---

# 18. Troubleshooting ImagePullBackOff

During the practical exercise, an invalid container image tag was used:

```yaml
image: lok2003/frontend-dashboard:v1.999
```

The tag did not exist in Docker Hub, so Kubernetes could not pull the image.

### Check Pod Status

```bash
kubectl get pods -n argocd-demo-2
```

Example:

```text
NAME                         READY   STATUS
httpd-demo-66675c995c-gszvm   0/1     ImagePullBackOff
```

### Inspect Pod Events

```bash
kubectl describe pod <pod-name> -n argocd-demo-2
```

Look at the Events section for errors such as:

```text
ErrImagePull
ImagePullBackOff
not found
```

### Understand the Error

- `ErrImagePull`: Kubernetes failed to pull the container image.
- `ImagePullBackOff`: Kubernetes is retrying the image pull with increasing delays.
- An incorrect image name, nonexistent tag, registry authentication problem, or network issue can cause image-pull failures.

### Fix the Image in Git

Restore a valid image in `deployment.yaml`:

```yaml
image: lok2003/frontend-dashboard:v1.23
```

Commit and push the correction:

```bash
git add deployment.yaml
git commit -m "Fix invalid image tag"
git push origin main
```

Check Argo CD:

```bash
kubectl get application nginx-demo-2 -n argocd
```

Check the Deployment:

```bash
kubectl rollout status deployment/httpd-demo -n argocd-demo-2
```

Check the Pods:

```bash
kubectl get pods -n argocd-demo-2
```

**Lesson learned:** Keep the image tag in Git valid. During a failed rolling update, Kubernetes may keep existing healthy Pods running while new Pods fail to start, but zero downtime is not guaranteed.

---

# 19. Argo CD Sync Status vs. Health Status

Argo CD reports synchronization and health separately.

### Sync Status

- `Synced`: Live resources match the desired configuration in Git.
- `OutOfSync`: Live resources differ from the desired configuration in Git.
- `Unknown`: Argo CD cannot determine the synchronization status.

### Health Status

- `Healthy`: Resources are operating as expected.
- `Progressing`: Resources are still being created, updated, or recovering.
- `Degraded`: One or more resources are unhealthy.
- `Missing`: An expected resource is missing.
- `Suspended`: A resource is suspended, where applicable.
- `Unknown`: Argo CD cannot determine the resource's health.

### Example from This Project

When the invalid image tag was pushed, Argo CD showed a synchronization status of `Synced` while the health status became unhealthy.

This happened because Git contained the same invalid image configuration that Kubernetes was trying to use.

After restoring the valid image and allowing the Deployment to recover, the application returned to:

```text
NAME           SYNC STATUS   HEALTH STATUS
nginx-demo-2   Synced        Healthy
```

**Key difference:**

- `Synced` means the live configuration matches Git.
- `Healthy` means the resources are operating as expected.

An application can be `Synced` but `Degraded`.

---

# 20. Understanding Argo CD Prune in Practice

The Argo CD Application uses:

```yaml
automated:
  prune: true
  selfHeal: true
```

### What Does Prune Do?

Pruning removes Argo CD-managed resources from Kubernetes when those resources are removed from the desired configuration in Git and the change is synchronized.

### Example: Removing a Service

The repository contains:

```text
argocd-nginx-demo-2/
├── application.yaml
├── deployment.yaml
├── service.yaml
└── README.md
```

The Service is named `nginx-service`.

Check the Service:

```bash
kubectl get service nginx-service -n argocd-demo-2
```

If `service.yaml` is removed from Git, committed, and pushed, Argo CD can delete the live Service during synchronization because pruning is enabled.

After deletion, this command:

```bash
kubectl get service nginx-service -n argocd-demo-2
```

returns a `NotFound` error.

### Are the Service Details Lost?

When the Service is deleted, its live configuration and ClusterIP are no longer available as a live Kubernetes resource.

However, the original manifest may still be recoverable from Git history if it was committed previously.

If the manifest is restored and synchronized, Kubernetes creates the Service again. The recreated Service generally receives a new ClusterIP unless a specific IP is configured and available.

**Important:** Pruning deletes resources; use it carefully in production environments.

---


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
