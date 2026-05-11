# ACM 2.16 App-of-Apps with OpenShift Virtualization (Pull Model)

This tutorial (WIP) demonstrates an **App-of-Apps pattern** using Red Hat Advanced Cluster Management (RHACM) 2.16 and OpenShift GitOps (ArgoCD) to deploy a VirtualMachine to a managed cluster via the **GitOps Pull Model**.  

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  Hub Cluster (local-cluster)                                        │
│                                                                     │
│  ┌──────────────────────┐     ┌──────────────────────────────────┐  │
│  │  Root ApplicationSet │────▶│  ArgoCD Application              │  │
│  │  (virt-root-appset)  │     │  (virt-child-appset-local-cluster│  │
│  │                      │     │   syncs child-appset/ directory) │  │
│  │  Generator:          │     └──────────────┬───────────────────┘  │
│  │   clusterDecision    │                    │                      │
│  │   (hub-local-cluster)│                    │                      │
│  └──────────────────────┘                    │ deploys              │
│                                              ▼                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ Child Resources (deployed to hub via root appset)             │  │
│  │                                                               │  │
│  │  • ManagedClusterSetBinding (binds cluster set to namespace)  │  │
│  │  • Placement            (selects managed clusters)            │  │
│  │  • GitOpsCluster        (creates cluster secrets for ArgoCD)  │  │
│  │  • Child ApplicationSet (pull model, targets managed clusters)│  │
│  └────────────────────────────────────────┬───────────────────────┘  │
│                                           │                         │
│               ManifestWork created        │                         │
│               per managed cluster         │                         │
└───────────────────────────────────────────┼─────────────────────────┘
                                            │
                          ┌─────────────────┘
                          ▼ (pull)
┌─────────────────────────────────────────────────────────────────────┐
│  Managed Cluster                                                    │
│                                                                     │
│  ┌──────────────────────┐     ┌──────────────────────────────────┐  │
│  │  Local ArgoCD        │────▶│  VirtualMachine                  │  │
│  │  (pulls Application  │     │  (fedora-gitops-demo)            │  │
│  │   from ManifestWork) │     │  deployed in virt-demo namespace │  │
│  └──────────────────────┘     └──────────────────────────────────┘  │
│                                                                     │
│  OpenShift Virtualization Operator installed                         │
└─────────────────────────────────────────────────────────────────────┘
```

## Directory Structure

```
appofappvirttest/
├── README.md
├── root-applicationset.yaml          # 1. Apply manually to hub (includes
│                                     #    ManagedClusterSetBinding, Placement
│                                     #    for local-cluster, GitOpsCluster, and
│                                     #    Root ApplicationSet with clusterDecision)
├── prerequisites/
│   └── managed-serviceaccount-setup.yaml  # Per-cluster: ManagedClusterAddOn,
│                                          # ManagedServiceAccount, ClusterPermission
├── child-appset/                     # 2. Synced to hub by root AppSet
│   ├── managedclustersetbinding.yaml
│   ├── placement.yaml
│   ├── gitopscluster.yaml            #    References managedServiceAccountRef
│   └── child-applicationset.yaml     #    Child AppSet (pull model)
├── vm-workload/                      # 3. Synced to managed clusters by child AppSet
│   ├── namespace.yaml                #    Includes argocd.argoproj.io/managed-by label
│   └── virtualmachine.yaml           #    Fedora VM deployed via GitOps
└── protectiontest.md                 # VM deletion protection guide (4 layers)
```

## Prerequisites

### Operators

Install the following on **both hub and all managed clusters**:

| Operator | Namespace | Purpose |
|----------|-----------|---------|
| Red Hat Advanced Cluster Management | `open-cluster-management` | Hub only — cluster management |
| Red Hat OpenShift GitOps | `openshift-gitops` | ArgoCD for GitOps workflows |
| OpenShift Virtualization (CNV) | `openshift-cnv` | KubeVirt — run VMs on OpenShift |

### Cluster Setup

1. **Hub cluster** has RHACM 2.16 installed and is the `local-cluster`.
2. **At least one managed cluster** is imported into RHACM.
3. **OpenShift GitOps Operator** (>= 1.9.0) is installed on both hub and managed clusters.
4. **OpenShift Virtualization Operator** is installed on both hub and managed clusters.
5. The managed cluster(s) are part of a `ManagedClusterSet` named `all-openshift-clusters`.

### Create the ManagedClusterSet (if not already existing)

```bash
oc apply -f - <<EOF
apiVersion: cluster.open-cluster-management.io/v1beta2
kind: ManagedClusterSet
metadata:
  name: all-openshift-clusters
EOF
```

Add `local-cluster` and your managed cluster(s) to the set:

```bash
oc label managedcluster local-cluster \
  cluster.open-cluster-management.io/clusterset=all-openshift-clusters --overwrite

oc label managedcluster <managed-cluster-name> \
  cluster.open-cluster-management.io/clusterset=all-openshift-clusters --overwrite
```

### Set Up Managed Service Account for Each Managed Cluster

The GitOpsCluster resource uses a `ManagedServiceAccount` to register managed clusters
with ArgoCD. Without this, ArgoCD cannot create cluster secrets and the pull model will
fail with permission errors.

For **each managed cluster**, apply the following on the Hub (replace `MANAGED_CLUSTER_NAME`
with the actual cluster namespace, e.g. `sno-2-xsr74`):

```bash
# Step 1: Enable the managed-serviceaccount add-on
oc apply -f - <<EOF
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: managed-serviceaccount
  namespace: MANAGED_CLUSTER_NAME
spec:
  installNamespace: open-cluster-management-agent-addon
EOF

# Step 2: Create the ManagedServiceAccount
oc apply -f - <<EOF
apiVersion: authentication.open-cluster-management.io/v1alpha1
kind: ManagedServiceAccount
metadata:
  name: argocd-manager
  namespace: MANAGED_CLUSTER_NAME
spec:
  rotation: {}
EOF

# Step 3: Grant cluster-admin permissions via ClusterPermission
oc apply -f - <<EOF
apiVersion: rbac.open-cluster-management.io/v1alpha1
kind: ClusterPermission
metadata:
  name: argocd-manager-permission
  namespace: MANAGED_CLUSTER_NAME
spec:
  clusterRoleBindings:
    - clusterRoleName: cluster-admin
      subject:
        kind: ServiceAccount
        name: argocd-manager
        namespace: open-cluster-management-managed-serviceaccount
EOF
```

A template YAML with all three resources is available at
[`prerequisites/managed-serviceaccount-setup.yaml`](prerequisites/managed-serviceaccount-setup.yaml).

The `GitOpsCluster` resource (in `child-appset/gitopscluster.yaml`) references this
service account via `managedServiceAccountRef: argocd-manager`. When the GitOps cluster
controller reconciles, it creates cluster secrets using the managed service account token
instead of the legacy secret method.

### ArgoCD RBAC for VirtualMachine Resources

The default ArgoCD application controller service account
(`openshift-gitops-argocd-application-controller`) does **not** have permission to create
`virtualmachines.kubevirt.io` resources. The `virt-demo` namespace includes the label
`argocd.argoproj.io/managed-by: openshift-gitops` (defined in `vm-workload/namespace.yaml`),
which triggers the GitOps Operator to automatically generate the localized RBAC permissions
the controller needs.

If you still encounter permission errors, grant broader access on the managed cluster:

```bash
oc adm policy add-cluster-role-to-user cluster-admin \
  system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller
```

---

## Step-by-Step Deployment

### Step 1: Verify Prerequisites

```bash
# Verify RHACM is running
oc get mch -A

# Verify OpenShift GitOps is running on the hub
oc get pods -n openshift-gitops

# Verify OpenShift Virtualization is installed on managed clusters
oc get csv -n openshift-cnv --context <managed-cluster-context>

# Verify managed clusters are imported
oc get managedclusters

# Verify cluster set membership
oc get managedclusters -l cluster.open-cluster-management.io/clusterset=all-openshift-clusters
```

### Step 2: Apply the Root ApplicationSet to the Hub

The root ApplicationSet file contains four resources that are applied together:
1. **ManagedClusterSetBinding** — binds `all-openshift-clusters` to the `openshift-gitops` namespace
2. **Placement** (`hub-local-cluster`) — selects only `local-cluster` via `clusterDecisionResource`
3. **GitOpsCluster** — registers `local-cluster` with the ArgoCD instance
4. **Root ApplicationSet** — uses `clusterDecisionResource` generator with the `hub-local-cluster` Placement

```bash
oc apply -f root-applicationset.yaml
```

This will:
- Create a Placement decision targeting `local-cluster`
- Create an ArgoCD Application named `virt-child-appset-local-cluster`
- Sync the contents of `child-appset/` to the `openshift-gitops` namespace on the hub

### Step 3: Verify the Root Resources and Child Resources

After applying, verify the root Placement decision and then the synced child resources:

```bash
# Verify the root Placement selected local-cluster
oc get placementdecisions -n openshift-gitops -l cluster.open-cluster-management.io/placement=hub-local-cluster

# Verify the root ApplicationSet created the ArgoCD Application
oc get applicationsets -n openshift-gitops
oc get applications -n openshift-gitops

# Verify the child resources were deployed
oc get managedclustersetbinding -n openshift-gitops
oc get placement -n openshift-gitops
oc get gitopscluster -n openshift-gitops

# Verify cluster secrets were created for managed clusters
oc get secrets -n openshift-gitops -l argocd.argoproj.io/secret-type=cluster
```

### Step 4: Verify the Pull Model Child ApplicationSet

The child ApplicationSet uses the **clusterDecisionResource generator** with OCM placement to discover managed clusters. It creates pull-model ArgoCD Applications for each.

```bash
# Verify the child ApplicationSet
oc get applicationset virt-vm-appset -n openshift-gitops -o yaml

# Verify Applications were generated for managed clusters
oc get applications -n openshift-gitops | grep virt-vm

# Check the MulticlusterApplicationSetReport
oc get multiclusterapplicationsetreport -n openshift-gitops
```

### Step 5: Verify the VirtualMachine on the Managed Cluster

The pull model controller creates a `ManifestWork` on the hub, which the managed cluster agent pulls. The local ArgoCD on the managed cluster then syncs the VM workload.

```bash
# On the hub: verify ManifestWork was created
oc get manifestwork -n <managed-cluster-name> | grep virt

# On the managed cluster: verify the VM is deployed
oc get virtualmachines -n virt-demo --context <managed-cluster-context>

# Check if the VM instance is running
oc get virtualmachineinstances -n virt-demo --context <managed-cluster-context>
```

### Step 6: Access the VirtualMachine

```bash
# Connect to the VM console via virtctl
virtctl console fedora-gitops-demo -n virt-demo --context <managed-cluster-context>

# Or via SSH (after obtaining the VM's IP)
virtctl ssh fedora@fedora-gitops-demo -n virt-demo --context <managed-cluster-context>
```

Default credentials: `fedora` / `fedora`

---

## How It Works

### Root ApplicationSet (clusterDecisionResource to Hub)

```
root-applicationset.yaml
    │
    ├── ManagedClusterSetBinding (all-openshift-clusters → openshift-gitops)
    ├── Placement: hub-local-cluster (selects only local-cluster)
    ├── GitOpsCluster: gitops-cluster-hub (registers local-cluster to ArgoCD)
    └── ApplicationSet: virt-root-appset
        ├── Generator: clusterDecisionResource (OCM Placement: hub-local-cluster)
        ├── Target: https://kubernetes.default.svc (hub itself)
        └── Source: child-appset/ directory in this repo
```

The root ApplicationSet uses a **clusterDecisionResource generator** referencing the `hub-local-cluster` Placement. This is consistent with the ACM pattern — both root and child use OCM Placement-based generators rather than mixing List and clusterDecision generators. The Placement selects only `local-cluster`, so one ArgoCD Application is created on the hub. This Application syncs the `child-appset/` directory, deploying the child ApplicationSet and its supporting resources.

### Child ApplicationSet (Pull Model to Managed Clusters)

```
child-appset/child-applicationset.yaml
    │
    ├── Generator: clusterDecisionResource (OCM Placement)
    ├── Placement: vm-managed-clusters (excludes local-cluster)
    ├── Pull Model Annotations:
    │   ├── argocd.argoproj.io/skip-reconcile: "true"
    │   ├── apps.open-cluster-management.io/ocm-managed-cluster: "{{name}}"
    │   └── apps.open-cluster-management.io/ocm-managed-cluster-app-namespace: openshift-gitops
    ├── Target: https://kubernetes.default.svc (local to each managed cluster)
    └── Source: vm-workload/ directory in this repo
```

Key aspects of the pull model:

| Setting | Value | Purpose |
|---------|-------|---------|
| `skip-reconcile` annotation | `"true"` | Prevents the hub ArgoCD from reconciling the Application |
| `ocm-managed-cluster` annotation | `"{{name}}"` | Identifies which managed cluster receives this Application |
| `destination.server` | `https://kubernetes.default.svc` | Ensures the managed cluster's local ArgoCD deploys the workload |
| `pull-to-ocm-managed-cluster` label | `"true"` | Enables the pull model controller to select this Application |

The RHACM pull controller wraps the generated Application into a `ManifestWork`, which the managed cluster's OCM agent pulls and applies locally. The local ArgoCD on the managed cluster then reconciles the Application by syncing directly from the Git repo.

### VM Workload

The `vm-workload/` directory contains a Fedora VirtualMachine using a `containerDisk` (no PVC provisioning required). The VM runs with:
- 1 CPU core, 1Gi memory
- Virtio disk and network interfaces
- Cloud-init for initial user setup
- Masquerade networking (pod network)

---

## Customization

### Target Specific Managed Clusters by Label

Edit `child-appset/placement.yaml` to select clusters by custom labels:

```yaml
spec:
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchLabels:
            environment: production
            has-cnv: "true"
```

### Use a Persistent VM with DataVolume

Replace the `containerDisk` in `vm-workload/virtualmachine.yaml` with a DataVolume for persistent storage:

```yaml
spec:
  dataVolumeTemplates:
    - metadata:
        name: fedora-gitops-demo-dv
      spec:
        sourceRef:
          kind: DataSource
          name: fedora
          namespace: openshift-virtualization-os-images
        storage:
          resources:
            requests:
              storage: 30Gi
  template:
    spec:
      volumes:
        - dataVolume:
            name: fedora-gitops-demo-dv
          name: rootdisk
```

### Add More Workloads

Add additional manifests (Services, Routes, etc.) to the `vm-workload/` directory. The child ApplicationSet will automatically sync them to all matched managed clusters.

---

## Troubleshooting

### Root ApplicationSet not syncing

```bash
# Check the root Placement is producing a decision
oc get placementdecisions -n openshift-gitops -l cluster.open-cluster-management.io/placement=hub-local-cluster -o yaml

# Verify the OCM placement generator ConfigMap exists
oc get configmap ocm-placement-generator -n openshift-gitops

# Check the ApplicationSet and Application status
oc get applicationset virt-root-appset -n openshift-gitops -o yaml
oc get application virt-child-appset-local-cluster -n openshift-gitops -o yaml
oc logs deployment/openshift-gitops-server -n openshift-gitops
```

### Child ApplicationSet not generating Applications

```bash
# Check placement decisions
oc get placementdecisions -n openshift-gitops

# Verify the OCM placement generator ConfigMap exists
oc get configmap ocm-placement-generator -n openshift-gitops

# Check GitOpsCluster status
oc get gitopscluster gitops-cluster-virt -n openshift-gitops -o yaml
```

### ManifestWork not created

```bash
# Verify the pull model controller is running
oc get pods -n open-cluster-management | grep pull

# Check ManifestWork on the hub
oc get manifestwork -A | grep virt
```

### ArgoCD sync fails with "virtualmachines.kubevirt.io is forbidden"

The ArgoCD application controller SA lacks permissions to create VMs:

```bash
# Option A: Verify the namespace has the managed-by label (triggers auto-RBAC)
oc get ns virt-demo -o jsonpath='{.metadata.labels.argocd\.argoproj\.io/managed-by}' --context <managed-cluster-context>

# Option B: Grant cluster-admin directly
oc adm policy add-cluster-role-to-user cluster-admin \
  system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller \
  --context <managed-cluster-context>
```

### Managed cluster not registered (no cluster secret)

```bash
# Verify the managed-serviceaccount add-on is available
oc get managedclusteraddon managed-serviceaccount -n <managed-cluster-name>

# Verify the ManagedServiceAccount exists and has a token
oc get managedserviceaccount argocd-manager -n <managed-cluster-name> -o yaml

# Verify the ClusterPermission exists
oc get clusterpermission argocd-manager-permission -n <managed-cluster-name>

# Check GitOpsCluster status
oc get gitopscluster gitops-cluster-virt -n openshift-gitops -o yaml
```

### VM not starting on managed cluster

```bash
# Check VM status
oc get vm fedora-gitops-demo -n virt-demo -o yaml --context <managed-cluster-context>

# Check VMI events
oc describe vmi fedora-gitops-demo -n virt-demo --context <managed-cluster-context>

# Verify CNV operator is healthy
oc get csv -n openshift-cnv --context <managed-cluster-context>
```

---

## Cleanup

```bash
# Remove the root ApplicationSet (cascading delete removes all child resources)
oc delete applicationset virt-root-appset -n openshift-gitops

# Verify cleanup on managed clusters
oc get vm -n virt-demo --context <managed-cluster-context>
oc get ns virt-demo --context <managed-cluster-context>
```

---

## References

- [RHACM 2.16 GitOps Documentation (PDF)](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.16/pdf/gitops/Red_Hat_Advanced_Cluster_Management_for_Kubernetes-2.16-GitOps-en-US.pdf)
- [Introducing the ArgoCD Pull Controller for RHACM](https://www.redhat.com/en/blog/introducing-the-argo-cd-application-pull-controller-for-red-hat-advanced-cluster-management)
- [OCM Application Lifecycle Management](https://open-cluster-management.io/docs/getting-started/integration/app-lifecycle/)
- [OCM ArgoCD Pull Model Getting Started](https://github.com/open-cluster-management-io/ocm/blob/main/solutions/deploy-argocd-apps-pull/getting-started.md)
- [OpenShift Virtualization Documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/virtualization/virtual-machines)
- [KubeVirt User Guide](https://kubevirt.io/user-guide/)
