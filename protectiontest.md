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

---

## Protection Layer 1: preserveResourcesOnDeletion on Root ApplicationSet

**Protects against:** Root ApplicationSet deletion cascading to child resources

**What to change in `root-applicationset.yaml`:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: virt-root-appset
  namespace: openshift-gitops
spec:
  generators:
    - list:
        elements:
          - cluster: local-cluster
            url: https://kubernetes.default.svc
  preservedFields:                          # ADD
    annotations:                            # ADD
      - argocd.argoproj.io/refresh          # ADD
  template:
    metadata:
      name: 'virt-child-appset-{{cluster}}'
    spec:
      project: default
      source:
        repoURL: 'https://github.com/ch-stark/appofappvirttest.git'
        targetRevision: main
        path: child-appset
      destination:
        namespace: openshift-gitops
        server: '{{url}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
        preserveResourcesOnDeletion: true   # ADD — child resources survive app deletion
```

---

## Protection Layer 2: preserveResourcesOnDeletion + prune:false on Child ApplicationSet

**Protects against:**
- Child ApplicationSet deletion cascading to VMs on managed clusters
- VM manifest removed from Git triggering VM deletion

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
  preservedFields:                          # ADD
    annotations:                            # ADD
      - argocd.argoproj.io/refresh          # ADD
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
        preserveResourcesOnDeletion: true   # ADD — VM survives app deletion
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
    - key: cluster.open-cluster-management.io/unavailable             # ADD
      operator: Exists                                                # ADD
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

**Note:** No `tolerationSeconds` is set — the cluster stays selected indefinitely.
Add `tolerationSeconds: 3600` if you want to de-select after 1 hour of downtime.

---

## Protection Layer 4: ArgoCD Delete=false on the VM Resource

**Protects against:** ArgoCD pruning or deleting the VM during any sync operation

This annotation on the VM itself tells ArgoCD to never delete this specific resource,
regardless of Application-level prune or delete settings.

**What to change in `vm-workload/virtualmachine.yaml`:**

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: fedora-gitops-demo
  namespace: virt-demo
  labels:
    app: fedora-gitops-demo
    app.kubernetes.io/managed-by: argocd
  annotations:
    argocd.argoproj.io/sync-options: Delete=false   # ADD — ArgoCD will never delete this
spec:
  running: true
  # ... rest unchanged
```

---

## Protection Layer 5: Block Namespace Deletion (ValidatingAdmissionPolicy)

**Protects against:** `oc delete ns virt-demo` cascade-deleting all resources including VMs

**Requires:** OpenShift 4.15+ (ValidatingAdmissionPolicy GA)
**Deploy to:** each managed cluster (directly or via ACM Policy)

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: deny-virt-namespace-deletion
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["namespaces"]
        operations: ["DELETE"]
  matchConditions:
    - name: only-virt-demo
      expression: "object.metadata.name == 'virt-demo'"
  validations:
    - expression: "false"
      message: >-
        Deletion of namespace 'virt-demo' is blocked to protect VirtualMachines.
        Remove this policy first if deletion is intentional.
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: deny-virt-namespace-deletion-binding
spec:
  policyName: deny-virt-namespace-deletion
  validationActions:
    - Deny
```

---

## Protection Layer 6: Block Direct VM Deletion (ValidatingAdmissionPolicy)

**Protects against:** `oc delete vm fedora-gitops-demo -n virt-demo` by any user

**Requires:** OpenShift 4.15+ (ValidatingAdmissionPolicy GA)
**Deploy to:** each managed cluster

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: deny-vm-deletion
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: ["kubevirt.io"]
        apiVersions: ["v1"]
        resources: ["virtualmachines"]
        operations: ["DELETE"]
  matchConditions:
    - name: only-protected-namespaces
      expression: "object.metadata.namespace == 'virt-demo'"
  validations:
    - expression: "request.userInfo.username == 'system:admin'"
      message: >-
        VirtualMachine deletion in namespace 'virt-demo' is blocked.
        Contact the platform team to remove this protection if intentional.
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: deny-vm-deletion-binding
spec:
  policyName: deny-vm-deletion
  validationActions:
    - Deny
```

---

## Protection Layer 7: ACM Policy — Last Resort Auto-Remediation

**Protects against:** VM somehow deleted despite all other protections

If the VM disappears for any reason, this ACM Policy detects it and recreates it.
Set `remediationAction: inform` if you only want alerting without auto-recreation.

**Deploy to:** hub cluster

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: vm-must-exist
  namespace: openshift-gitops
  annotations:
    policy.open-cluster-management.io/standards: NIST SP 800-53
    policy.open-cluster-management.io/categories: CM Configuration Management
    policy.open-cluster-management.io/controls: CM-2 Baseline Configuration
spec:
  remediationAction: enforce
  disabled: false
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: vm-must-exist
        spec:
          remediationAction: enforce
          severity: critical
          object-templates:
            - complianceType: musthave
              objectDefinition:
                apiVersion: kubevirt.io/v1
                kind: VirtualMachine
                metadata:
                  name: fedora-gitops-demo
                  namespace: virt-demo
                spec:
                  running: true
---
apiVersion: policy.open-cluster-management.io/v1
kind: PlacementBinding
metadata:
  name: vm-must-exist-binding
  namespace: openshift-gitops
spec:
  placementRef:
    apiGroup: cluster.open-cluster-management.io
    kind: Placement
    name: vm-managed-clusters
  subjects:
    - apiGroup: policy.open-cluster-management.io
      kind: Policy
      name: vm-must-exist
```

---

## Summary

| Layer | What | Protects Against | Recovery Time |
|-------|------|-----------------|---------------|
| 1 | `preserveResourcesOnDeletion` on root | Root AppSet deletion cascade | Instant (resources left in place) |
| 2 | `preserveResourcesOnDeletion` + `prune: false` on child | Child AppSet deletion cascade + Git removal | Instant (VM never touched) |
| 3 | Placement tolerations | Cluster unreachable/unavailable | Instant (cluster stays selected) |
| 4 | `Delete=false` on VM | ArgoCD prune/sync deleting VM | Instant (ArgoCD skips deletion) |
| 5 | ValidatingAdmissionPolicy on namespace | `oc delete ns virt-demo` | Instant (API rejects the call) |
| 6 | ValidatingAdmissionPolicy on VM | `oc delete vm` by any user | Instant (API rejects the call) |
| 7 | ACM Policy enforce | Any deletion that bypasses above | Seconds (policy controller recreates) |

**Recommended minimum:** Layers 1 + 2 + 3 + 4 (all ArgoCD/Placement level, no extra operators needed).

**Full protection:** All 7 layers for defense in depth.
