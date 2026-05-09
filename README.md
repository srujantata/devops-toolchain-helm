# devops-toolchain-helm

![Helm](https://img.shields.io/badge/Helm-3.14-0F1689?logo=helm)
![ArgoCD](https://img.shields.io/badge/ArgoCD-latest-EF7B4D)
![License](https://img.shields.io/badge/license-MIT-blue)

Helm values and ArgoCD App-of-Apps manifests for deploying a full DevOps toolchain onto EKS.

## Tools Deployed

| Tool | Chart | Purpose |
|------|-------|---------|
| Jenkins | jenkins/jenkins | CI/CD orchestration |
| Bitbucket DC | atlassian/bitbucket | Source control (HA) |
| JFrog Artifactory HA | jfrog/artifactory-ha | Artifact repository |
| SonarQube Community | sonarqube/sonarqube | Code quality |
| Harbor | harbor/harbor | Container registry |

## Skills Demonstrated

`Helm` · `ArgoCD` · `GitOps` · `Jenkins` · `JFrog` · `SonarQube` · `Kubernetes`
