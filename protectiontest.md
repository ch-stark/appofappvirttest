# VM Deletion Protection — Defense in Depth

## The Problem: Cascade Deletion Chains

### Chain 1: Root ApplicationSet deleted

```
Delete Root ApplicationSet
  → deletes ArgoCD Application (virt-child-appset-local-cluster)
    → cascade-deletes child resources (child AppSet, Placement, GitOpsCluster)
      → child ApplicationSet deletion deletes generated Applications
        → each Application cascade-deletes managed resources
          → VM is gone, namespace is gone — disaster
```

### Chain 2: Namespace deleted

```
Delete namespace virt-demo
  → Kubernetes garbage-collects ALL namespace-scoped resources
    → VM is gone (ValidatingAdmissionPolicy on the VM alone won't help)
```

### Chain 3: Managed cluster temporarily unreachable

```
Managed cluster loses connectivity
  → ACM taints cluster with unreachable/unavailable
    → Placement de-selects the cluster (no toleration)
      → ApplicationSet removes the Application for that cluster
        → cascade-delete of VM workload
```

### Chain 4: Hub cluster temporarily unavailable (e.g. during upgrade)

```
Hub cluster upgrade begins
  → ACM controllers, ArgoCD AppSet controller, pull model controller go down
    → Hub comes back, sees stale state, re-evaluates Placements
      → Managed clusters may appear unreachable during hub restart window
        → Placement de-selects clusters before connectivity fully restores
          → ApplicationSet removes Applications → cascade-delete of VMs
```

### The Toleration Timeout Dilemma

Setting Placement tolerations **without a timeout** (indefinite) protects VMs during
upgrades and outages, but introduces a different problem:

```
Managed cluster permanently decommissioned or truly dead
  → Cluster stays selected in Placement forever (tolerated indefinitely)
    → ApplicationSet keeps generating Applications for a dead cluster
      → ManifestWorks pile up with no agent to consume them
        → MulticlusterApplicationSetReport shows permanent degraded state
          → No alert fires — the dead cluster is silently tolerated
            → Operators lose visibility into real cluster health
```

**The tradeoff:**

| tolerationSeconds | Risk if too short | Risk if too long / infinite |
|-------------------|-------------------|-----------------------------|
| None (no toleration) | Network blip deletes VMs | — |
| 1800 (30 min) | Longer outage or slow upgrade deletes VMs | — |
| 14400 (4 hours) | — | Dead cluster hidden for 4 hours |
| No timeout (infinite) | — | Dead clusters never cleaned up, silent failures accumulate |

**Recommendation:** Use a timeout that covers the longest expected disruption
(hub upgrade + spoke upgrade can take 2-4 hours), but is not infinite.
Combined with `preserveResourcesOnDeletion: true` on the child ApplicationSet,
even if the timeout expires and the Application is removed, the VM stays running.

```
tolerationSeconds: 14400 (4 hours)
  + preserveResourcesOnDeletion: true
  = VM survives even if tolerance expires
  + dead clusters are eventually cleaned up from Placement
```

---

## Protection Layer 1: preserveResourcesOnDeletion on Root ApplicationSet

**Protects against:** Root ApplicationSet deletion cascading to child resources

There are **two levels** of `preserveResourcesOnDeletion` and both are needed:

| Level | Location | What it preserves |
|-------|----------|-------------------|
| ApplicationSet-level | `spec.syncPolicy.preserveResourcesOnDeletion` | Generated **Applications** survive when the **ApplicationSet** is deleted |
| Template-level | `spec.template.spec.syncPolicy.preserveResourcesOnDeletion` | Managed **resources** survive when the **Application** is deleted |

Without the ApplicationSet-level setting, deleting the ApplicationSet still deletes all
generated Applications — even if the template-level setting would have kept the resources.
The cascade never reaches the template-level check because the Applications are gone first.

**What to change in `root-applicationset.yaml`:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: virt-root-appset
  namespace: openshift-gitops
spec:
  generators:
    - clusterDecisionResource:
        configMapRef: ocm-placement-generator
        labelSelector:
          matchLabels:
            cluster.open-cluster-management.io/placement: hub-local-cluster
        requeueAfterSeconds: 30
  syncPolicy:
    preserveResourcesOnDeletion: true       # ADD — generated Applications survive AppSet deletion
  template:
    metadata:
      name: 'virt-child-appset-{{name}}'
    spec:
      project: default
      source:
        repoURL: 'https://github.com/ch-stark/appofappvirttest.git'
        targetRevision: main
        path: child-appset
      destination:
        namespace: openshift-gitops
        server: 'https://kubernetes.default.svc'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
        preserveResourcesOnDeletion: true   # ADD — child resources survive Application deletion
```

---

## Protection Layer 2: preserveResourcesOnDeletion + prune:false on Child ApplicationSet

**Protects against:**
- Child ApplicationSet deletion cascading to VMs on managed clusters
- VM manifest removed from Git triggering VM deletion

Same two-level pattern as Layer 1:
- **ApplicationSet-level:** `spec.syncPolicy.preserveResourcesOnDeletion` — generated Applications survive child AppSet deletion
- **Template-level:** `spec.template.spec.syncPolicy.preserveResourcesOnDeletion` — VMs survive Application deletion

**What to change in `child-appset/child-applicationset.yaml`:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: virt-vm-appset
  namespace: openshift-gitops
spec:
  generators:
    - clusterDecisionResource:
        configMapRef: ocm-placement-generator
        labelSelector:
          matchLabels:
            cluster.open-cluster-management.io/placement: vm-managed-clusters
        requeueAfterSeconds: 30
  syncPolicy:
    preserveResourcesOnDeletion: true       # ADD — generated Applications survive AppSet deletion
  template:
    metadata:
      name: 'virt-vm-{{name}}'
      labels:
        apps.open-cluster-management.io/pull-to-ocm-managed-cluster: 'true'
      annotations:
        argocd.argoproj.io/skip-reconcile: 'true'
        apps.open-cluster-management.io/ocm-managed-cluster: '{{name}}'
        apps.open-cluster-management.io/ocm-managed-cluster-app-namespace: openshift-gitops
    spec:
      project: default
      source:
        repoURL: 'https://github.com/ch-stark/appofappvirttest.git'
        targetRevision: main
        path: vm-workload
      destination:
        namespace: virt-demo
        server: 'https://kubernetes.default.svc'
      syncPolicy:
        automated:
          prune: false                      # CHANGE — don't delete resources removed from Git
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
        preserveResourcesOnDeletion: true   # ADD — VMs survive Application deletion
```

---

## Protection Layer 3: Placement Tolerations

**Protects against:** Temporary cluster connectivity loss triggering VM deletion

Without tolerations, when a managed cluster goes temporarily unreachable, ACM taints it.
The Placement de-selects the cluster, the ApplicationSet removes the Application, and the
VM gets cascade-deleted — even though the VM is still running fine on the managed cluster.

**What to change in `child-appset/placement.yaml`:**

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: vm-managed-clusters
  namespace: openshift-gitops
spec:
  tolerations:                                                        # ADD
    - key: cluster.open-cluster-management.io/unreachable             # ADD
      operator: Exists                                                # ADD
      tolerationSeconds: 14400                                        # ADD — 4 hours, covers upgrade windows
    - key: cluster.open-cluster-management.io/unavailable             # ADD
      operator: Exists                                                # ADD
      tolerationSeconds: 14400                                        # ADD — 4 hours, covers upgrade windows
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: vendor
              operator: "In"
              values:
                - OpenShift
            - key: name
              operator: NotIn
              values:
                - local-cluster
```

**Why 14400 seconds (4 hours)?** A full OpenShift upgrade cycle (hub + spoke) can take
2-4 hours. Setting the timeout to 4 hours covers this window. After 4 hours, if the
cluster is still unreachable, the Placement de-selects it — but `preserveResourcesOnDeletion`
on the child ApplicationSet (Layer 2) ensures the VM keeps running regardless.

**Why not infinite?** Indefinite tolerations hide permanently dead clusters. You lose
visibility into real cluster health, ManifestWorks accumulate for non-existent agents,
and the MulticlusterApplicationSetReport shows permanent degraded state with no alert.

**The safety net:** Even if the toleration expires and the Application is removed,
`preserveResourcesOnDeletion: true` (Layer 2) prevents the VM from being deleted.
The two layers work together: tolerations buy time, preserveResourcesOnDeletion
guarantees the VM survives no matter what.

---

## Protection Layer 4: Block All Deletions in the VM Namespace (ValidatingAdmissionPolicy)

**Protects against:** Deletion of the namespace, the VM, or any of its related resources

A VM depends on many related resources. Protecting only the VM itself is not enough:

| Resource | What happens if deleted |
|----------|----------------------|
| **Namespace** | Kubernetes garbage-collects everything — VM, PVCs, Secrets, all gone |
| **PersistentVolumeClaim / DataVolume** | VM loses its disk — data gone, VM crashes or fails to start |
| **Secret** | VM loses SSH keys, cloud-init userdata, or TLS certs |
| **ConfigMap** | VM loses configuration data |
| **Service / Route** | VM becomes unreachable over the network |
| **NetworkAttachmentDefinition** | VM loses secondary network interfaces |
| **VirtualMachineInstancetype / Preference** | New VMs can't be created with the same profile |

A single policy that blocks ALL deletions in the namespace is simpler and more
comprehensive than individual policies per resource type.

**Requires:** OpenShift 4.15+ (ValidatingAdmissionPolicy GA)
**Deploy to:** each managed cluster (directly or via ACM Policy)

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: deny-virt-demo-deletions
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: ["*"]
        apiVersions: ["*"]
        resources: ["*"]
        operations: ["DELETE"]
  matchConditions:
    - name: only-virt-demo-namespace
      expression: >-
        (has(object.metadata.namespace) && object.metadata.namespace == 'virt-demo')
        || (object.kind == 'Namespace' && object.metadata.name == 'virt-demo')
  validations:
    - expression: "request.userInfo.username == 'system:admin'"
      message: >-
        All deletions in namespace 'virt-demo' are blocked to protect
        VirtualMachines and their related resources. Only system:admin
        can delete resources here. Contact the platform team if intentional.
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: deny-virt-demo-deletions-binding
spec:
  policyName: deny-virt-demo-deletions
  validationActions:
    - Deny
```

This single policy blocks:
- `oc delete ns virt-demo` — namespace deletion (cascade protection)
- `oc delete vm <name> -n virt-demo` — direct VM deletion
- `oc delete pvc <name> -n virt-demo` — disk storage deletion
- `oc delete secret <name> -n virt-demo` — credential/key deletion
- Any other resource deletion in the namespace

Only `system:admin` can bypass. Adjust the expression to allow specific
ServiceAccounts if needed (e.g., ArgoCD for managed sync operations).

---

## Summary

| Layer | What | Protects Against | Recovery Time |
|-------|------|-----------------|---------------|
| 1 | `preserveResourcesOnDeletion` on root (both AppSet-level + template-level) | Root AppSet deletion cascade | Instant (Applications + resources left in place) |
| 2 | `preserveResourcesOnDeletion` on child (both levels) + `prune: false` | Child AppSet deletion cascade + Git removal | Instant (Applications + VMs left in place) |
| 3 | Placement tolerations (4h timeout) | Cluster unreachable/unavailable + hub upgrade | Instant (cluster stays selected for 4h, then Layer 2 takes over) |
| 4 | ValidatingAdmissionPolicy on entire namespace | `oc delete` of namespace, VM, PVC, Secret, or any resource | Instant (API rejects the call) |

**Recommended minimum:** Layers 1 + 2 + 3 (ArgoCD/Placement level, no extra operators needed).

**Full protection:** All 4 layers for defense in depth.
