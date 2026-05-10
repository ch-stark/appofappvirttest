# Resilience at Scale: Architecting Disaster Recovery for Argo CD and RHACM
 
When managing tens of thousands of resources—such as virtual machines—via GitOps, the **blast radius** of a control plane failure becomes the primary concern.
 
In the Red Hat Advanced Cluster Management (RHACM) and Argo CD pull model, the Hub cluster acts as the brain. If that brain loses its connection or its state, managed clusters may interpret the silence as a command to "clean up" (delete) their workloads.
 
To prevent a cascade deletion during a Hub failure or upgrade, a multi-layered defense strategy is required—one that prioritizes architectural stability over reactive automation.
 
## Layer 1: Ensuring Stability in the Hub-to-Cluster Relationship
 
In the pull model, the Hub uses `Placements` to determine which managed clusters should receive an `Application`.
 
If the Hub loses connectivity to a managed cluster (for example during a Hub upgrade or a network partition), it marks that cluster with `unreachable` or `unavailable` taints.
 
Without proper configuration, the Placement engine may immediately evict the managed cluster from the decision-making process. This can trigger a chain reaction:
 
1. The `ApplicationSet` detects the cluster is gone
2. The child `Application` is removed from the Hub
3. Argo CD may prune and delete workloads from the managed cluster
### The Steady-State Placement Strategy
 
Instead of allowing the Hub to react to transient health issues, apply tolerations so that Placements ignore temporary connectivity gaps.
 
This ensures the Hub's intended state remains intact while the environment is unstable.
 
```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: app-placement
spec:
  numberOfClusters: 1
  tolerations:
    - key: cluster.open-cluster-management.io/unreachable
      operator: Exists
    - key: cluster.open-cluster-management.io/unavailable
      operator: Exists
```
 
By omitting a timeout, the Hub maintains its source of truth indefinitely.
 
During a Disaster Recovery restore, the Hub may require time to re-establish inventory and cluster connectivity. This configuration prevents it from issuing mass-deletion commands during that critical rediscovery phase.
 
## Layer 2: Breaking the Cascade (Delete Protection)
 
The ultimate safety net is decoupling the lifecycle of the Hub's configuration from the lifecycle of the actual workloads.
 
Even if a Placement miscalculates or an `ApplicationSet` disappears during restore operations, the physical resources—such as virtual machines—must remain protected.
 
### Resource Preservation Strategy
 
Set the following on the `ApplicationSet`:
 
```yaml
preserveResourcesOnDeletion: true
```
 
This ensures that if an `Application` is deleted from the Hub, the workloads on the managed cluster are orphaned but preserved.
 
This is arguably the single most effective protection against catastrophic workload deletion during:
 
- Control plane migrations
- Hub upgrades
- DR failovers
- Accidental `ApplicationSet` removal
 
## Layer 3: The Passive Hub "Warm" Standby
 
A true Disaster Recovery strategy begins long before the incident occurs.
 
A passive Hub cluster should already exist, pre-provisioned with the same operator stack:
 
- RHACM
- OpenShift GitOps
- Required backup and restore tooling
### Intelligent Backup Selection
 
While the ACM Backup Operator automatically handles many RHACM resources, several core Kubernetes and Argo CD resources require explicit labeling to ensure inclusion in OADP/Velero backups.
 
| Resource Type | Backup Status | Action Required |
|---|---|---|
| `ApplicationSet` / `GitOpsCluster` | Automatic | None |
| `ArgoCD` / `ManagedServiceAccount` | Automatic | None |
| Repository & Cluster Secrets | Manual | Label with `cluster.open-cluster-management.io/backup=argocd` |
| Argo CD ConfigMaps | Manual | Label with `cluster.open-cluster-management.io/backup=argocd` |
 
 
## Layer 4: Preventing Split-Brain via Governance
 
In an Active-Passive topology, preventing both Hubs from simultaneously managing clusters is critical.
 
A recommended approach is to enforce a **deny SyncWindow** policy on the Passive Hub using RHACM Governance Policies.


The "Safety Switch": Global Deny Policy
This RHACM Governance policy uses a ConfigurationPolicy to dynamically find every AppProject on the passive Hub and inject a 24/7 deny window.

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: block-argocd-syncs-passive-hub
  namespace: open-cluster-management
spec:
  disabled: false
  remediationAction: enforce  # This will actively overwrite projects to stay locked
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: deny-argocd-syncwindow
        spec:
          remediationAction: enforce
          severity: high
          object-templates-raw: |
            {{- range (lookup "argoproj.io/v1alpha1" "AppProject" "openshift-gitops" "").items }}
            - complianceType: musthave
              objectDefinition:
                apiVersion: argoproj.io/v1alpha1
                kind: AppProject
                metadata:
                  name: {{ .metadata.name }}
                  namespace: openshift-gitops
                spec:
                  syncWindows:
                    - kind: deny
                      schedule: "* * * * *"
                      duration: 24h
                      applications: ["*"]
                      namespaces: ["*"]
                      clusters: ["*"]
            {{- end }}
```

This effectively "locks" the secondary Hub until administrators intentionally promote it during failover.
 
### The Failover Sequence
 
When promoting the Secondary Hub, the order of operations is essential.
 
#### 1. Isolate the Primary Hub
 
Confirm the Primary Hub is completely offline to avoid conflicting GitOps actions and split-brain behavior.
 
#### 2. Unlock the Secondary Hub
 
Disable the Governance Policy enforcing the Argo CD sync block.
 
#### 3. Execute the Restore
 
Run the `Restore` custom resource. Using:
 
```yaml
cleanupBeforeRestore: CleanupRestored
```
 
This ensures stale data from previous DR tests does not interfere with the production recovery.
 
#### 4. Verify Pull Reconnection
 
Once the following reconnect:
 
- `ManagedServiceAccount` tokens
- `GitOpsCluster` resources
the Argo CD dashboards will naturally repopulate without manual intervention.
 
 
## Summary
 
At scale, resilience is not simply about recovering infrastructure—it is about ensuring the recovery process itself does not amplify the outage.
 
When managing thousands of resources through GitOps, Disaster Recovery must account for:
 
- Transient connectivity loss
- Control plane instability
- Backup consistency
- Split-brain prevention
- Safe workload preservation
By combining **Placement tolerations**, **resource preservation**, **managed backups**, and **governance-based failover controls**, you create a GitOps platform capable of surviving even severe control-plane disruptions without triggering catastrophic cascade deletions.
