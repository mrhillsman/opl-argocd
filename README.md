# OpenShift Cluster Provisioning with ArgoCD, ACM, and Hive

GitOps-based automation for provisioning and managing OpenShift clusters on AWS using Red Hat Advanced Cluster Management (ACM) and Hive.

## Overview

This repository provides a declarative, GitOps approach to OpenShift cluster lifecycle management:

- **ArgoCD**: Continuous deployment and cluster configuration management
- **ACM (Advanced Cluster Management)**: Multi-cluster management and governance
- **Hive**: Automated OpenShift cluster provisioning on cloud platforms

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Hub Cluster                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────────┐  │
│  │          │  │          │  │                          │  │
│  │  ArgoCD  │─▶│   ACM    │─▶│  Hive Controllers        │  │
│  │          │  │          │  │                          │  │
│  └──────────┘  └──────────┘  └──────────────────────────┘  │
│       │              │                     │                │
└───────┼──────────────┼─────────────────────┼────────────────┘
        │              │                     │
        │ (GitOps)     │ (Import)            │ (Provision)
        │              │                     │
        ▼              ▼                     ▼
   ┌─────────┐   ┌──────────┐        ┌──────────┐
   │   Git   │   │ Managed  │        │   AWS    │
   │  Repo   │   │ Clusters │        │          │
   └─────────┘   └──────────┘        └──────────┘
```

## Directory Structure

```
.
├── bootstrap/                          # Bootstrap ArgoCD resources
│   ├── argocd-project.yaml            # ArgoCD AppProject
│   └── argocd-app-of-apps.yaml        # App-of-Apps pattern
├── cluster-templates/                  # Reusable cluster templates
│   └── aws-ha/                        # AWS HA cluster template
│       ├── base/                      # Base Kustomize resources
│       │   ├── kustomization.yaml
│       │   ├── namespace.yaml
│       │   ├── clusterdeployment.yaml
│       │   ├── machinepool-worker.yaml
│       │   ├── managedcluster.yaml
│       │   └── install-config-secret.yaml
│       └── README.md
├── clusters/                          # Cluster instances
│   ├── dev-cluster-01/
│   │   ├── kustomization.yaml        # Cluster-specific overlay
│   │   ├── cluster-config.yaml       # Configuration parameters
│   │   └── argocd-application.yaml   # ArgoCD Application
│   └── prod-cluster-01/
│       ├── kustomization.yaml
│       ├── cluster-config.yaml
│       └── argocd-application.yaml
├── secrets/
│   └── README.md                      # Secrets management guide
└── README.md
```

## Prerequisites

### Hub Cluster Requirements

1. **OpenShift 4.12+** cluster (hub cluster)
2. **Red Hat Advanced Cluster Management (ACM) 2.8+** installed
3. **OpenShift GitOps (ArgoCD)** installed
4. **Hive** operator (typically installed with ACM)

### Access Requirements

1. **AWS Account** with appropriate permissions
2. **Red Hat pull secret** from console.redhat.com
3. **Git repository** access (this repo)
4. **DNS domain** configured in AWS Route53

### Install Prerequisites

```bash
# Install OpenShift GitOps operator
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: stable
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Install ACM operator
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: advanced-cluster-management
  namespace: open-cluster-management
spec:
  channel: release-2.9
  name: advanced-cluster-management
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Wait for operators to be ready
oc wait --for=condition=Ready pods -l control-plane=controller-manager -n openshift-gitops --timeout=300s
oc wait --for=condition=Ready pods -l app=multiclusterhub-operator -n open-cluster-management --timeout=300s
```

## Quick Start

### 1. Clone this repository

```bash
git clone https://github.com/your-org/opl-argocd.git
cd opl-argocd
```

### 2. Update repository URLs

Update the `repoURL` fields in:
- [bootstrap/argocd-app-of-apps.yaml](bootstrap/argocd-app-of-apps.yaml)
- [clusters/dev-cluster-01/argocd-application.yaml](clusters/dev-cluster-01/argocd-application.yaml)
- [clusters/prod-cluster-01/argocd-application.yaml](clusters/prod-cluster-01/argocd-application.yaml)

### 3. Configure secrets

Create required secrets for cluster provisioning. See [secrets/README.md](secrets/README.md) for detailed instructions.

**IMPORTANT**: These secrets must be created in the cluster namespace BEFORE ArgoCD syncs the cluster resources, or recreated if the namespace gets deleted during sync operations.

```bash
# Example: Create secrets for dev-cluster-01

# 1. Create namespace (ArgoCD can also create this, but creating it manually ensures secrets persist)
oc create namespace dev-cluster-01

# 2. AWS credentials
oc create secret generic aws-credentials \
  --from-literal=aws_access_key_id=YOUR_AWS_ACCESS_KEY \
  --from-literal=aws_secret_access_key=YOUR_AWS_SECRET_KEY \
  -n dev-cluster-01

# 3. Pull secret (download from console.redhat.com)
oc create secret generic pull-secret \
  --from-file=.dockerconfigjson=pull-secret.json \
  --type=kubernetes.io/dockerconfigjson \
  -n dev-cluster-01

# 4. SSH key (generate new or use existing)
ssh-keygen -t rsa -b 4096 -f dev-cluster-01-ssh-key -N ""
oc create secret generic dev-cluster-01-ssh-key \
  --from-file=ssh-privatekey=dev-cluster-01-ssh-key \
  --from-file=ssh-publickey=dev-cluster-01-ssh-key.pub \
  --type=kubernetes.io/ssh-auth \
  -n dev-cluster-01
```

**Note**: Hive automatically injects `pullSecret` and `sshKey` into the install-config from the ClusterDeployment's secret references. Do not add them manually to the install-config.yaml.

### 4. Deploy ArgoCD project

```bash
# Apply the AppProject and App-of-Apps
oc apply -f bootstrap/

# Verify ArgoCD project is created
oc get appproject cluster-provisioning -n openshift-gitops

# Verify App-of-Apps is syncing
oc get application cluster-provisioning-apps -n openshift-gitops
```

### 5. Provision a cluster

The cluster will be automatically provisioned when ArgoCD syncs. Monitor the progress:

```bash
# Watch ClusterDeployment status
oc get clusterdeployment -n dev-cluster-01 -w

# Check ClusterProvision resource
oc get clusterprovision -n dev-cluster-01

# View detailed status
oc describe clusterdeployment dev-cluster-01 -n dev-cluster-01

# Monitor provisioning pod logs (find pod name first)
oc get pods -n dev-cluster-01 | grep provision
oc logs -f <provision-pod-name> -n dev-cluster-01
```

Provisioning typically takes 30-45 minutes.

**Key Status Indicators:**
- `PROVISIONSTATUS: Provisioning` - Cluster is being created
- `PROVISIONSTATUS: Provisioned` - Cluster is ready
- `INFRAID` is populated - AWS resources are being created
- Provision pod is `Running` - Installation in progress
- Provision pod shows `Error` - Check logs for specific errors

### 6. Access the provisioned cluster

```bash
# Extract admin kubeconfig
oc extract secret/dev-cluster-01-admin-kubeconfig -n dev-cluster-01 --to=-

# Or save to file
oc extract secret/dev-cluster-01-admin-kubeconfig -n dev-cluster-01 --to=. --confirm

# Use the kubeconfig
export KUBECONFIG=./kubeconfig
oc get nodes
```

## Important Operational Notes

### Namespace Management

**ArgoCD may delete and recreate namespaces during sync operations**, especially if sync fails. This will delete all secrets in the namespace. To avoid issues:

1. **Create the namespace manually first** before triggering ArgoCD sync
2. **Keep secret files** (pull-secret.json, SSH keys, AWS credentials) accessible to recreate them if needed
3. **Cancel stuck sync operations** if the namespace keeps getting deleted:
   ```bash
   oc patch application <cluster-name> -n openshift-gitops --type merge -p '{"operation":null}'
   ```

### ArgoCD RBAC Requirements

The ArgoCD application controller needs cluster-admin permissions to create Hive and ACM resources. This is configured in the bootstrap via:

```bash
# This ClusterRoleBinding is required for cluster provisioning
oc create clusterrolebinding openshift-gitops-cluster-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=openshift-gitops:openshift-gitops-argocd-application-controller
```

### OpenShift Version Selection

Check available ClusterImageSets on your hub cluster:

```bash
oc get clusterimageset
```

The hub cluster must have ClusterImageSets for the OpenShift versions you want to deploy. Update your cluster configuration to use an available version:

```yaml
# In cluster-config.yaml
openshiftVersion: img4.20.10-x86-64-appsub  # Use actual ClusterImageSet name
```

### Secret Recreation After Namespace Deletion

If the namespace was deleted during ArgoCD sync, you must recreate secrets:

```bash
# 1. Wait for namespace to be stable
oc create namespace dev-cluster-01 2>/dev/null || echo "Namespace exists"

# 2. Recreate AWS credentials
oc create secret generic aws-credentials \
  --from-literal=aws_access_key_id=YOUR_KEY \
  --from-literal=aws_secret_access_key=YOUR_SECRET \
  -n dev-cluster-01

# 3. Recreate pull-secret
oc create secret generic pull-secret \
  --from-file=.dockerconfigjson=pull-secret.json \
  --type=kubernetes.io/dockerconfigjson \
  -n dev-cluster-01

# 4. Recreate SSH key
oc create secret generic dev-cluster-01-ssh-key \
  --from-file=ssh-privatekey=dev-cluster-01-ssh-key \
  --from-file=ssh-publickey=dev-cluster-01-ssh-key.pub \
  --type=kubernetes.io/ssh-auth \
  -n dev-cluster-01

# 5. Manually apply cluster resources if ArgoCD sync is problematic
oc kustomize clusters/dev-cluster-01 | oc apply -f -
```

### Manual Intervention Workflow

If ArgoCD sync is failing or causing namespace deletion loops:

```bash
# 1. Disable automated sync
oc patch application <cluster-name> -n openshift-gitops \
  --type merge -p '{"spec":{"syncPolicy":{"automated":null}}}'

# 2. Cancel any running operations
oc patch application <cluster-name> -n openshift-gitops \
  --type merge -p '{"operation":null}'

# 3. Ensure namespace exists
oc create namespace <cluster-name>

# 4. Create/verify secrets exist (see above)
oc get secrets -n <cluster-name>

# 5. Manually apply resources
oc kustomize clusters/<cluster-name> | oc apply -f -

# 6. Monitor provision pod
oc get pods -n <cluster-name> -w
```

## Adding a New Cluster

### Using the template

1. **Create a new cluster directory**:
   ```bash
   mkdir -p clusters/my-new-cluster
   ```

2. **Copy template files**:
   ```bash
   cp clusters/dev-cluster-01/kustomization.yaml clusters/my-new-cluster/
   cp clusters/dev-cluster-01/cluster-config.yaml clusters/my-new-cluster/
   cp clusters/dev-cluster-01/argocd-application.yaml clusters/my-new-cluster/
   ```

3. **Customize cluster configuration** in `clusters/my-new-cluster/cluster-config.yaml`:
   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: cluster-config
     namespace: my-new-cluster
   data:
     clusterName: my-new-cluster
     baseDomain: my-domain.com
     region: us-east-1
     openshiftVersion: openshift-v4.14.0
     environment: development

     controlPlaneInstanceType: m5.xlarge
     controlPlaneReplicas: "3"

     workerInstanceType: m5.2xlarge
     workerReplicas: "3"
     workerZones: "us-east-1a,us-east-1b,us-east-1c"
   ```

4. **Update kustomization.yaml** to reference correct cluster name

5. **Update ArgoCD application** with new cluster path

6. **Create secrets** for the new cluster namespace

7. **Commit and push**:
   ```bash
   git add clusters/my-new-cluster/
   git commit -m "Add my-new-cluster configuration"
   git push
   ```

ArgoCD will automatically detect and deploy the new cluster.

## Cluster Lifecycle Management

### Scaling Workers

Update the `workerReplicas` in `cluster-config.yaml` and commit:

```yaml
workerReplicas: "6"  # Scale to 6 workers
```

### Upgrading OpenShift Version

1. Ensure ClusterImageSet exists for target version:
   ```bash
   oc get clusterimageset
   ```

2. Update `openshiftVersion` in cluster-config.yaml:
   ```yaml
   openshiftVersion: openshift-v4.15.0
   ```

3. Commit and let ArgoCD sync

### Hibernating Clusters (Cost Savings)

```bash
# Hibernate a cluster
oc patch clusterdeployment dev-cluster-01 -n dev-cluster-01 \
  --type merge -p '{"spec":{"powerState":"Hibernating"}}'

# Resume a cluster
oc patch clusterdeployment dev-cluster-01 -n dev-cluster-01 \
  --type merge -p '{"spec":{"powerState":"Running"}}'
```

### Deprovisioning a Cluster

**WARNING**: This will permanently delete the cluster and all data.

1. Set `preserveOnDelete: false` in the ClusterDeployment
2. Delete the ArgoCD Application:
   ```bash
   oc delete application dev-cluster-01 -n openshift-gitops
   ```
3. Or delete the ClusterDeployment directly:
   ```bash
   oc delete clusterdeployment dev-cluster-01 -n dev-cluster-01
   ```

## Customization

### Supported Parameters

Each cluster can customize:
- Cluster name and base domain
- AWS region and availability zones
- OpenShift version (via ClusterImageSet)
- Control plane instance type and count
- Worker instance type and count
- Network CIDRs (pod, service, machine)

See [cluster-templates/aws-ha/README.md](cluster-templates/aws-ha/README.md) for detailed customization options.

### Adding Custom Manifests

Add custom manifests to be installed during cluster provisioning:

1. Create a ConfigMap with your manifests
2. Reference it in ClusterDeployment:
   ```yaml
   spec:
     provisioning:
       manifestsConfigMapRef:
         name: my-custom-manifests
   ```

## Troubleshooting

### Cluster stuck in "Provisioning"

```bash
# Check ClusterDeployment conditions
oc get clusterdeployment <cluster-name> -n <namespace> -o yaml

# View provision pod logs (not job)
oc get pods -n <namespace> | grep provision
oc logs -f <provision-pod-name> -n <namespace>

# Check ClusterProvision status
oc get clusterprovision -n <namespace>
oc describe clusterprovision <provision-name> -n <namespace>

# Check for AWS API errors
oc describe clusterdeployment <cluster-name> -n <namespace>
```

### Common Provisioning Errors

#### Error: "ssh: no key found" or "Invalid value: ssh-rsa AAAA...placeholder"

**Cause**: Install-config contains placeholder SSH key or pullSecret instead of letting Hive inject them.

**Solution**: Ensure install-config.yaml does NOT contain `sshKey` or `pullSecret` fields. Hive injects these automatically from ClusterDeployment secret references.

#### Error: "ManagedCluster validation webhook denied: url "" is invalid"

**Cause**: ManagedCluster has empty or invalid API server URL in `managedClusterClientConfigs`.

**Solution**: Remove the `managedClusterClientConfigs` field from ManagedCluster spec. ACM will populate it automatically after cluster is provisioned.

#### Error: "Hive controller panic: nil pointer dereference" in retrofitMetadataJSON

**Cause**: ClusterDeployment has `clusterMetadata` fields (adminKubeconfigSecretRef, adminPasswordSecretRef, clusterID, infraID) set before provisioning.

**Solution**: Remove `clusterMetadata` section from ClusterDeployment template. Hive populates these fields automatically during provisioning.

#### Error: "ClusterImageSet not found"

**Cause**: Referenced OpenShift version doesn't have a ClusterImageSet on the hub cluster.

**Solution**:
```bash
# Check available versions
oc get clusterimageset

# Update cluster-config.yaml and kustomization.yaml with available version
openshiftVersion: img4.20.10-x86-64-appsub
```

### ArgoCD sync failures

```bash
# Check Application status
oc get application -n openshift-gitops

# View Application events
oc describe application <cluster-name> -n openshift-gitops

# Check sync operation details
oc get application <cluster-name> -n openshift-gitops -o jsonpath='{.status.operationState}'

# Check ArgoCD logs
oc logs -n openshift-gitops deployment/openshift-gitops-application-controller
```

#### Error: "Application resource not permitted in project"

**Cause**: ArgoCD project lacks permission for Application resources (needed for app-of-apps pattern).

**Solution**: Add Application to namespaceResourceWhitelist in ArgoCD project:
```yaml
namespaceResourceWhitelist:
  - group: argoproj.io
    kind: Application
```

#### Error: "Hive/ACM resources not permitted in project"

**Cause**: ArgoCD project doesn't allow Hive and ACM CRDs.

**Solution**: Ensure bootstrap/argocd-project.yaml includes all required resource types in both `clusterResourceWhitelist` and `namespaceResourceWhitelist`.

### Namespace Keeps Getting Deleted

**Cause**: ArgoCD sync failures trigger namespace deletion/recreation cycles.

**Solution**:
1. Cancel the sync operation: `oc patch application <cluster-name> -n openshift-gitops --type merge -p '{"operation":null}'`
2. Disable automated sync: `oc patch application <cluster-name> -n openshift-gitops --type merge -p '{"spec":{"syncPolicy":{"automated":null}}}'`
3. Create namespace manually: `oc create namespace <cluster-name>`
4. Recreate secrets (see "Secret Recreation" section above)
5. Manually apply resources: `oc kustomize clusters/<cluster-name> | oc apply -f -`

### Cluster not importing to ACM

```bash
# Check ManagedCluster status
oc get managedcluster

# Verify import secret
oc get secret -n <cluster-name> <cluster-name>-import

# Check ManagedCluster conditions
oc get managedcluster <cluster-name> -o yaml

# Check ACM klusterlet status on provisioned cluster
export KUBECONFIG=<cluster-kubeconfig>
oc get klusterlet -A
```

## Security Considerations

1. **Secrets**: Use Sealed Secrets or External Secrets Operator for production
2. **RBAC**: Restrict ArgoCD project permissions to specific namespaces
3. **AWS Credentials**: Use IAM roles with minimal permissions
4. **Network Policies**: Implement network segmentation between clusters
5. **Git Access**: Use SSH keys or tokens with read-only access where possible

See [secrets/README.md](secrets/README.md) for detailed secrets management best practices.

## Best Practices

1. **Use separate AWS accounts** for dev/staging/prod clusters
2. **Enable cluster hibernation** for non-production environments
3. **Set preserveOnDelete: true** to prevent accidental cluster deletion
4. **Use ClusterSets** in ACM to organize clusters by environment
5. **Implement backup strategy** for cluster configurations and etcd
6. **Monitor cluster costs** using AWS Cost Explorer tags
7. **Use ApplicationSets** when scaling to many similar clusters

## Advanced Features

### Using ApplicationSets for Multi-Cluster

For managing many similar clusters, consider using ArgoCD ApplicationSets:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-provisioning-set
spec:
  generators:
    - git:
        repoURL: https://github.com/your-org/opl-argocd.git
        revision: main
        directories:
          - path: clusters/*
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: cluster-provisioning
      source:
        repoURL: https://github.com/your-org/opl-argocd.git
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
```

### ClusterPools for Faster Provisioning

Pre-provision clusters for faster availability:

```yaml
apiVersion: hive.openshift.io/v1
kind: ClusterPool
metadata:
  name: aws-dev-pool
spec:
  baseDomain: dev.example.com
  imageSetRef:
    name: openshift-v4.14.0
  platform:
    aws:
      credentialsSecretRef:
        name: aws-credentials
      region: us-east-1
  pullSecretRef:
    name: pull-secret
  size: 2  # Keep 2 clusters ready
```

## Contributing

1. Fork this repository
2. Create a feature branch
3. Make your changes
4. Test with a development cluster
5. Submit a pull request

## Resources

- [OpenShift Hive Documentation](https://github.com/openshift/hive/tree/master/docs)
- [Red Hat ACM Documentation](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/)
- [OpenShift GitOps Documentation](https://docs.openshift.com/container-platform/latest/cicd/gitops/understanding-openshift-gitops.html)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Kustomize Documentation](https://kubectl.docs.kubernetes.io/references/kustomize/)

## License

MIT License - See LICENSE file for details
