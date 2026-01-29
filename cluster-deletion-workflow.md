# Cluster Deletion Workflow

This diagram visualizes the complete cluster deletion process from trigger to completion.

```mermaid
sequenceDiagram
    autonumber
    participant User
    participant Git
    participant ArgoCD as ArgoCD Controller
    participant K8s as Kubernetes API
    participant Hive as Hive Controller
    participant AWS
    participant CronJob as Cleanup CronJob

    Note over User,CronJob: Phase 1: Deletion Trigger

    User->>Git: Remove clusters/<cluster>/argocd-application.yaml
    User->>Git: git commit && git push
    Git-->>ArgoCD: Webhook/Poll detects change

    Note over User,CronJob: Phase 2: ArgoCD Application Deletion

    ArgoCD->>ArgoCD: App-of-Apps detects missing Application manifest
    ArgoCD->>K8s: Delete Application: <cluster-name>
    K8s-->>ArgoCD: Application deletion initiated

    Note over User,CronJob: Phase 3: Resource Deletion (Reverse Sync-Wave Order)

    ArgoCD->>K8s: Delete KlusterletAddonConfig (sync-wave: 2)
    ArgoCD->>K8s: Delete ManagedCluster (sync-wave: 1)
    ArgoCD->>K8s: Delete MachinePool (sync-wave: 0)
    ArgoCD->>K8s: Delete ClusterDeployment (sync-wave: 0)

    Note over User,CronJob: Phase 4: Hive Uninstall Process

    K8s-->>Hive: ClusterDeployment deletion detected
    Hive->>Hive: Check for ClusterDeprovision
    Hive->>K8s: Create ClusterDeprovision resource
    Hive->>K8s: Create Job: <cluster>-uninstall

    Note over Hive,AWS: Uninstall job requires metadata-json secret

    Hive->>K8s: Read Secret: <cluster>-metadata-json
    K8s-->>Hive: Return infraID, clusterID, AWS metadata

    Note over User,CronJob: Phase 5: AWS Resource Cleanup (10-15 minutes)

    Hive->>AWS: Delete EC2 instances
    Hive->>AWS: Delete Load Balancers
    Hive->>AWS: Delete Security Groups
    Hive->>AWS: Delete VPC resources
    Hive->>AWS: Delete Route53 records
    Hive->>AWS: Delete S3 buckets
    Hive->>AWS: Delete EBS volumes
    Hive->>AWS: Delete IAM resources
    AWS-->>Hive: Resources deleted

    Hive->>K8s: Update Job status: Complete
    Hive->>K8s: Delete ClusterDeprovision

    Note over User,CronJob: Phase 6: Secret Protection (Finalizers)

    rect rgb(255, 240, 240)
        Note over K8s: Secrets have finalizer: openshiftpartnerlabs.com/deprovision
        K8s->>K8s: Attempt delete Secret: aws-credentials
        K8s->>K8s: Blocked by finalizer
        K8s->>K8s: Attempt delete Secret: <cluster>-metadata-json
        K8s->>K8s: Blocked by finalizer
        Note over K8s: Secrets remain until finalizers removed
    end

    Note over User,CronJob: Phase 7: Cleanup CronJob (runs every 5 minutes)

    CronJob->>K8s: Query secrets with label: deprovision-pending=true
    K8s-->>CronJob: Return: aws-credentials, <cluster>-metadata-json

    loop For each labeled secret
        CronJob->>K8s: Get Job: <cluster>-uninstall status
        K8s-->>CronJob: status.conditions[Complete]=True
        CronJob->>CronJob: Uninstall complete, safe to cleanup

        CronJob->>K8s: Remove finalizer from secret
        K8s-->>CronJob: Finalizer removed
        CronJob->>K8s: Remove label: deprovision-pending
        K8s-->>CronJob: Label removed
    end

    Note over User,CronJob: Phase 8: Final Cleanup

    K8s->>K8s: Secret: aws-credentials deleted (no finalizer)
    K8s->>K8s: Secret: <cluster>-metadata-json deleted (no finalizer)
    K8s->>K8s: Secret: <cluster>-admin-kubeconfig deleted
    K8s->>K8s: Secret: <cluster>-admin-password deleted

    ArgoCD->>K8s: Delete remaining resources
    ArgoCD->>K8s: Delete Namespace: <cluster-name>
    K8s-->>ArgoCD: Namespace deleted

    Note over User,CronJob: Deletion Complete
```

## Key Components

| Component | Role |
|-----------|------|
| **Git Repository** | Source of truth for cluster state |
| **ArgoCD App-of-Apps** | Discovers and manages cluster Applications |
| **ArgoCD Controller** | Syncs desired state, handles deletion |
| **Hive Controller** | Manages ClusterDeployment lifecycle |
| **Uninstall Job** | Destroys AWS infrastructure |
| **Cleanup CronJob** | Removes finalizers after uninstall completes |

## Finalizer Protection

The finalizers on `aws-credentials` and `<cluster>-metadata-json` secrets ensure:

1. Secrets are not deleted before uninstall job runs
2. Uninstall job has access to AWS credentials and cluster metadata
3. AWS resources are properly cleaned up before secrets are removed

## Timing

| Phase | Duration |
|-------|----------|
| ArgoCD detection | 1-3 minutes |
| Resource deletion trigger | < 1 minute |
| AWS cleanup | 10-15 minutes |
| CronJob cleanup | Up to 5 minutes (next scheduled run) |
| **Total** | **15-25 minutes** |
