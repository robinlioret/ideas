# High Level Design

## General Workflow

```mermaid
flowchart LR
    user01@{ shape: circle, label: "User" }
    git-config@{ shape: rounded, label: "<i>git</i><br/>User configurations" }
    argocd@{ shape: subproc, label: "ArgoCD" }
    k8s-stack@{ shape: card, label: "Pulumi Stack" }
    pulumi-operator@{ shape: subproc, label: "Pulumi<br/>Operator" }
    pulumi-job@{ shape: processes, label: "Pulumi<br/>Stack Job" }
    git-iac@{ shape: rounded, label: "<i>git</i><br/>IaC Code" }
    secrets-store@{ shape: rounded, label: "Secret store" }
    user02@{ shape: circle, label: "User" }
    admin01@{ shape: circle, label: "Admin" }
    resources@{ shape: docs, label: "Resources" }
    
    user01 -->|commit| git-config
    argocd -->|pull| git-config
    argocd -->|deploy| k8s-stack
    admin01 -->|maintain| git-iac
    pulumi-operator -->|control| k8s-stack
    pulumi-operator -->|create| pulumi-job
    pulumi-job -->|pull| git-iac
    pulumi-job -->|manage| resources
    pulumi-job -->|pull| secrets-store
    user02 -->|populate| secrets-store
    
```

The workflow (general idea):

1. The user sets the secrets in the secret store.
2. The user commits changes to the repository (CRUD operations.)
3. ArgoCD pulls the wanted state.
4. ArgoCD generates the Kubernetes resources with a helm chart.
5. The Pulumi Operator detect the changes
6. The Pulumi Operator creates a job for the stack.
7. The Pulumi Stack Job pulls the IaC module.
8. The Pulumi Stack Job pulls the secrets.
9. The Pulumi Stack Job deploys the stack's resources.
10. ArgoCD detects the stack status and send a notification to the user.