# devops-toolchain-helm

![Helm](https://img.shields.io/badge/Helm-3.14-0F1689?logo=helm)
![ArgoCD](https://img.shields.io/badge/GitOps-ArgoCD-EF7B4D?logo=argo)
![Jenkins](https://img.shields.io/badge/Jenkins-2.x-D24939?logo=jenkins)
![SonarQube](https://img.shields.io/badge/SonarQube-2026.x-4E9BCD?logo=sonarqube)
![Harbor](https://img.shields.io/badge/Harbor-2.15.0-60B932?logo=harbor)
![License](https://img.shields.io/badge/license-MIT-blue)

> Helm values and ArgoCD App-of-Apps manifests for deploying a full DevOps toolchain on AWS EKS.
> Cluster: `devops-poc` (us-east-1) — provisioned by [aws-eks-platform](https://github.com/srujantata/aws-eks-platform).

---

## Live Tools (POC — us-east-1)

| Tool | Version | URL | Login | Status |
|------|---------|-----|-------|--------|
| **ArgoCD** | argo-cd 9.x | http://a944fe58b20d24057b8cf0af7f586c3a-245014201.us-east-1.elb.amazonaws.com | admin / kubectl secret | ✅ Live |
| **Jenkins** | jenkins 5.9.22 | http://a910314d11cee49a99b923e221108e05-468852420.us-east-1.elb.amazonaws.com:8080 | admin / `s3TMPc1dfyrWzAj9zLyGgx` | ✅ Live |
| **SonarQube** | sonarqube 2026.x | http://a386f7224766d43d6b50aeee825abe34-1292105297.us-east-1.elb.amazonaws.com:9000 | admin / admin | ✅ Live |
| **Harbor** | harbor 1.19.0 | http://a68756541aa5a454d81e6515e061e3c2-574233704.us-east-1.elb.amazonaws.com | admin / Harbor12345 | ✅ Live |

---

## Repository Structure

```
devops-toolchain-helm/
│
├── charts/
│   ├── jenkins/
│   │   └── values.yaml       ← Jenkins Helm overrides
│   ├── sonarqube/
│   │   └── values.yaml       ← SonarQube community edition overrides
│   └── harbor/
│       └── values.yaml       ← Harbor container registry overrides
│
└── README.md
```

---

## Tools — Config Highlights

### ArgoCD — GitOps Controller

Manages all other tools via App-of-Apps pattern. Any change to this repo auto-syncs to the cluster.

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd --create-namespace

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

---

### Jenkins — CI/CD

Chart: `jenkins/jenkins 5.9.22`

Key values (`charts/jenkins/values.yaml`):

| Setting | Value | Why |
|---------|-------|-----|
| `serviceType` | `LoadBalancer` | External access via AWS Classic LB |
| `persistence.size` | `5Gi` | EBS gp3 volume for Jenkins home |
| `persistence.storageClass` | `ebs-gp3` | Custom StorageClass from aws-eks-platform |
| `resources.requests` | `200m CPU / 512Mi RAM` | Fits on t3.medium alongside other tools |
| `resources.limits` | `500m CPU / 1Gi RAM` | Prevents runaway builds starving other pods |
| `numExecutors` | `2` | Two concurrent build slots per pod |

```bash
helm repo add jenkins https://charts.jenkins.io
helm install jenkins jenkins/jenkins -n jenkins --create-namespace \
  -f charts/jenkins/values.yaml

# Get admin password
kubectl exec -n jenkins svc/jenkins -c jenkins -- \
  /bin/cat /run/secrets/additional/chart-admin-password
```

**Access:** http://JENKINS_LB_HOSTNAME:8080

---

### SonarQube — Code Quality

Chart: `sonarqube/sonarqube 2026.x (Community Edition)`

Key values (`charts/sonarqube/values.yaml`):

| Setting | Value | Why |
|---------|-------|-----|
| `community.enabled` | `true` | Community edition flag (changed in 2026.x chart) |
| `postgresql.enabled` | `false` | Embedded H2 database for POC (no extra pod) |
| `jvmOpts` | `-Xmx512m -Xms256m` | Constrains ES8 + JVM to fit in t3.medium |
| `resources.requests` | `300m CPU / 1Gi RAM` | SonarQube 2026.x minimum (OOMKills below 1.6GB total) |
| `resources.limits` | `1000m CPU / 2Gi RAM` | Hard ceiling |
| `serviceType` | `LoadBalancer` | External access |

```bash
helm repo add sonarqube https://SonarSource.github.io/helm-chart-sonarqube
helm install sonarqube sonarqube/sonarqube -n sonarqube --create-namespace \
  -f charts/sonarqube/values.yaml \
  --set monitoringPasscode=admin123
```

**Access:** http://SONARQUBE_LB_HOSTNAME:9000  
**Default login:** admin / admin (change on first login)

> **POC note:** For production, set `postgresql.enabled=true` or point at Aurora PostgreSQL. H2 is not supported for production use.

---

### Harbor — Container Registry

Chart: `harbor/harbor 1.19.0` (Harbor App 2.15.0)

Key values (`charts/harbor/values.yaml`):

| Setting | Value | Why |
|---------|-------|-----|
| `expose.type` | `loadBalancer` | External access via AWS Classic LB |
| `expose.tls.enabled` | `false` | POC — no cert-manager configured |
| `persistence.enabled` | `true` | EBS gp3 volumes for registry + database + Redis |
| `trivy.enabled` | `true` | Built-in CVE scanning on every image push |
| `resources.requests` | `100m CPU / 256Mi RAM` per component | Fits t3.medium (8 pods total) |

```bash
helm repo add harbor https://helm.goharbor.io
helm install harbor harbor/harbor -n harbor --create-namespace \
  -f charts/harbor/values.yaml
```

**Pods deployed (all 8 Running):**

| Pod | Purpose |
|-----|---------|
| `harbor-core` | API server + auth |
| `harbor-portal` | Web UI |
| `harbor-nginx` | Reverse proxy / ingress |
| `harbor-registry` | OCI registry storage (2 containers) |
| `harbor-jobservice` | Async job queue (replication, GC) |
| `harbor-database-0` | PostgreSQL |
| `harbor-redis-0` | Session + cache |
| `harbor-trivy-0` | CVE scanner |

**Access:** http://HARBOR_LB_HOSTNAME  
**Default login:** admin / Harbor12345

> **Production note:** Set `expose.tls.enabled=true` with cert-manager and update `externalURL` to your Route53 hostname. Without this, `docker push` redirect URLs will use the internal hostname.

---

## Install All Tools (One Block)

```bash
# Add all Helm repos
helm repo add argo       https://argoproj.github.io/argo-helm
helm repo add jenkins    https://charts.jenkins.io
helm repo add sonarqube  https://SonarSource.github.io/helm-chart-sonarqube
helm repo add harbor     https://helm.goharbor.io
helm repo update

# Deploy
helm install argocd    argo/argo-cd              -n argocd    --create-namespace
helm install jenkins   jenkins/jenkins           -n jenkins   --create-namespace -f charts/jenkins/values.yaml
helm install sonarqube sonarqube/sonarqube       -n sonarqube --create-namespace -f charts/sonarqube/values.yaml --set monitoringPasscode=admin123
helm install harbor    harbor/harbor             -n harbor    --create-namespace -f charts/harbor/values.yaml

# Verify
kubectl get pods -A
kubectl get svc  -A | grep LoadBalancer
```

---

## Uninstall All Tools (Before Terraform Destroy)

```bash
# Uninstall in reverse order (removes LoadBalancers from AWS billing)
helm uninstall harbor    -n harbor
helm uninstall sonarqube -n sonarqube
helm uninstall jenkins   -n jenkins
helm uninstall argocd    -n argocd

# Delete PVCs (releases EBS volumes — saves ~$5/month)
kubectl delete pvc --all -n jenkins
kubectl delete pvc --all -n sonarqube
kubectl delete pvc --all -n harbor

# Wait for ALBs to deregister before terraform destroy
Start-Sleep -Seconds 120
```

---

## Resource Requirements

All 4 tools fit on 2× t3.medium nodes (4 vCPU, 8GB RAM total):

| Tool | CPU Request | RAM Request | RAM Limit | PVC |
|------|------------|-------------|-----------|-----|
| Jenkins | 200m | 512Mi | 1Gi | 5Gi |
| SonarQube | 300m | 1Gi | 2Gi | 5Gi |
| Harbor (all pods) | ~600m | ~1.5Gi | ~3Gi | 15Gi |
| ArgoCD | ~100m | ~256Mi | 512Mi | — |
| **Total** | **~1.2 CPU** | **~3.3Gi** | **~6.5Gi** | **~25Gi** |

> t3.medium = 2 vCPU / 4GB RAM. Two nodes give 4 vCPU / 8GB — enough for all tools + system pods.
> **t3.small will OOMKill SonarQube** — it needs 1.6GB free just for ES8 + JVM.

---

## Bugs Fixed During Deployment

| Tool | Bug | Fix |
|------|-----|-----|
| SonarQube | OOMKill on every start | Upgraded nodes t3.small → t3.medium |
| SonarQube | `Unknown field: edition` chart error | `edition=community` removed in 2026.x — use `community.enabled=true` |
| SonarQube | Liveness probe failing | `monitoringPasscode` now required in newer chart versions |
| Harbor | `jobservice` 2 restarts on init | Known startup race (jobservice waits for core) — self-resolves, cosmetic |

---

## Related Repos

| Repo | Purpose |
|------|---------|
| [aws-eks-platform](https://github.com/srujantata/aws-eks-platform) | Terraform — EKS cluster, VPC, EBS CSI, StorageClass |
| [infra-prompt-engine](https://github.com/srujantata/infra-prompt-engine) | AI prompt → Terraform/kubectl admin |
| [github-actions-iac](https://github.com/srujantata/github-actions-iac) | CI/CD workflows + OIDC auth |
| [dr-failover-runbook](https://github.com/srujantata/dr-failover-runbook) | Active-passive DR runbook |

---

## Skills Demonstrated

`Helm` · `ArgoCD` · `GitOps` · `App-of-Apps` · `Jenkins` · `SonarQube` · `Harbor` · `Trivy` · `Kubernetes` · `EBS CSI` · `DevSecOps` · `Container Registry` · `CI/CD`
