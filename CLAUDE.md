# devops-toolchain-helm — LLM Context

## What This Repo Does
Helm values files and ArgoCD manifests for the DevOps toolchain running on EKS. ArgoCD App-of-Apps pattern bootstraps all tools from a single root application.

## Tools Deployed
| Tool | Helm Repo | Chart | Namespace | Purpose |
|------|-----------|-------|-----------|---------|
| Jenkins | `https://charts.jenkins.io` | `jenkins/jenkins` | `jenkins` | CI/CD orchestration |
| Bitbucket DC | `https://atlassian.github.io/data-center-helm-charts` | `atlassian/bitbucket` | `bitbucket` | Source control (HA) |
| JFrog Artifactory HA | `https://charts.jfrog.io` | `jfrog/artifactory-ha` | `artifactory` | Artifact repository |
| SonarQube | `https://SonarSource.github.io/helm-chart-sonarqube` | `sonarqube/sonarqube` | `sonarqube` | Code quality |
| Harbor | `https://helm.goharbor.io` | `harbor/harbor` | `harbor` | Container registry |

## Directory Structure
```
argocd/
├── bootstrap/
│   └── app-of-apps.yaml    ← Root ArgoCD Application — apply this first
└── apps/
    ├── jenkins.yaml         ← ArgoCD Application for Jenkins
    ├── bitbucket.yaml
    ├── artifactory.yaml
    ├── sonarqube.yaml
    └── harbor.yaml

charts/
├── jenkins/values.yaml      ← Helm overrides for Jenkins
├── bitbucket/values.yaml
├── artifactory/values.yaml
├── sonarqube/values.yaml
└── harbor/values.yaml
```

## Key Conventions
- **Node affinity**: all tool pods use `nodeSelector: {role: tools}` — on On-Demand nodes, never Spot
- **Storage**: EBS gp3 for single-pod tools (SonarQube, Jenkins), EFS for multi-replica (Bitbucket)
- **Secrets**: ALL credentials come from AWS Secrets Manager via External Secrets Operator (ESO) — no secrets in this repo
- **DB**: Aurora PostgreSQL (external) for Bitbucket, Artifactory, SonarQube, Harbor
- **POC simplification**: single replica, reduced resource requests, SQLite/embedded DB where possible

## Bootstrap ArgoCD (run after EKS is up)
```bash
# Install ArgoCD
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd --create-namespace

# Apply root app-of-apps — deploys all tools automatically
kubectl apply -f argocd/bootstrap/app-of-apps.yaml
```

## POC vs Production Values
| Setting | POC | Production |
|---------|-----|-----------|
| Jenkins replicas | 1 | 2 |
| Artifactory chart | `artifactory` (single) | `artifactory-ha` (3 nodes) |
| Bitbucket replicas | 1 | 3 |
| Database | Embedded / SQLite | Aurora PostgreSQL (external RDS) |
| Storage | 10Gi EBS | 100–500Gi EBS/EFS |
| Resource requests | 256m CPU / 512Mi | 1-2 CPU / 4-8Gi |

## What NOT to Do
- Never put Helm passwords or license keys in values.yaml — use ESO ExternalSecret references
- Never use `latest` image tag — pin to specific versions for reproducibility
- Never deploy Bitbucket Server (end-of-life) — use Bitbucket Data Center chart
- Don't modify ArgoCD sync policy to manual without good reason — selfHeal keeps drift corrected

## Related Repos
- `aws-eks-platform` — provides the EKS cluster and ArgoCD installation
- `dr-failover-runbook` — DR-region ArgoCD ApplicationSet for standby cluster
