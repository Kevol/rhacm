# Advanced Cluster Management (ACM) Cleanup & Troubleshooting Guide

This guide compiles the operational steps required to resolve stuck uninstallation states, admission webhook blockages, stuck finalizers, and standalone deployment conflicts when managing or wiping out Red Hat Advanced Cluster Management (ACM) / Open Cluster Management on OpenShift.

---

## Phase 1: Handling Webhook Blockers (Initial Deletion Errors)

When deleting `MulticlusterHub` components, you may encounter an `InternalError` indicating that the validating webhook cannot be reached (`service "multiclusterhub-operator-webhook" not found`). This happens because the API server is configured to validate requests using a service that has already been torn down.

### 1. Remove the Validating Webhook Configuration
Bypass the blocker by deleting the cluster-scoped validating webhook configuration:
```bash
oc delete validatingwebhookconfiguration multiclusterhub.validating-webhook.open-cluster-management.io
```

### 2. Remove the Companion Mutating Webhook Configuration
To prevent a symmetric error right after, clear the mutating webhook configuration:
```bash
oc delete mutatingwebhookconfiguration multiclusterhub.mutating-webhook.open-cluster-management.io
```

### 3. Retry the Component Deletion
Once the API plane no longer looks for the missing webhook service, execute the deletion again:
```bash
oc delete multiclusterhub --all
```

---

## Phase 2: Clearing Stuck Resource Finalizers

If the resource deletion command hangs indefinitely, the `MulticlusterHub` object is waiting on background finalization logic (`finalizer.operator.open-cluster-management.io`) that cannot complete.

### 1. Check for Active Finalizers
Verify what finalizers are currently attached to the resource:
```bash
oc get multiclusterhub multiclusterhub -n open-cluster-management -o jsonpath='{.metadata.finalizers}'
```

### 2. Force Clear via JSON Patch (Recommended)
Bypass the controller queue and strip the finalizer array directly via a strict JSON patch:
```bash
oc patch multiclusterhub multiclusterhub -n open-cluster-management --type=json -p '[{"op": "remove", "path": "/metadata/finalizers"}]'
```

### 3. Alternative: Manual Live Edit
If the patch command fails or is overridden by a running controller status update, open the live manifest manually:
```bash
oc edit multiclusterhub multiclusterhub -n open-cluster-management
```
Find the `finalizers:` block under `metadata:`, delete the lines entirely, save, and exit.

---

## Phase 3: Purging and Wiping the Namespace

To entirely remove the `open-cluster-management` namespace when it becomes stuck in a `Terminating` status due to lingering cluster-scoped custom resource dependencies.

### 1. Initiate Namespace Deletion
```bash
oc delete namespace open-cluster-management
```

### 2. Bulk Clean Remaining Labeled Components & Sub-resource Finalizers
If the namespace hangs, clear all remaining webhook resources matching the operator labels and programmatically patch any custom resources left behind inside the namespace:
```bash
# Delete all webhooks matching the operator app label
oc delete mutatingwebhookconfiguration -l app=multiclusterhub-operator
oc delete validatingwebhookconfiguration -l app=multiclusterhub-operator

# Loop through all namespaced API resources and strip remaining finalizers inside the target namespace
for res in $(oc api-resources --namespaced=true --verbs=delete -o name); do
    oc get $res -n open-cluster-management -o jsonpath='{.items[*].metadata.name}' | xargs -r -n 1 oc patch -n open-cluster-management $res --type=merge -p '{"metadata":{"finalizers":null}}' 2>/dev/null
done
```

### 3. Check for Broken APIServices
Stale APIService registrations can cause namespace deletion routines to stall. Check for failing API endpoints:
```bash
oc get apiservice | grep False
```
If an APIService tied to `open-cluster-management` returns `False`, remove it:
```bash
oc delete apiservice <broken-apiservice-name>
```

### 4. The Ultimate Escape Hatch (Bypass Control Plane)
If the namespace remains stuck caching old errors despite the above, force-clear its finalizers directly through the cluster raw API server endpoint:
```bash
oc get namespace open-cluster-management -o json | jq '.spec.finalizers = []' > ns-clean.json
oc replace --raw "/api/v1/namespaces/open-cluster-management/finalize" -f ns-clean.json
```
*(Note: If `jq` is unavailable, manually edit `ns-clean.json` to change `"finalizers": [ "kubernetes" ]` to `"finalizers": []` before executing the `oc replace` command).*

---

## Phase 4: Resolving Standalone Mode Constraints

When redeploying or applying a new configuration, you may encounter an admission webhook denial: `MultiClusterHub in Standalone mode already exists: multiclusterhub`. This occurs because ACM strictly enforces a global limit of one `MulticlusterHub` instance per cluster.

### 1. Locate the Lingering Instance
Search across all cluster namespaces to find where the conflicting instance is currently hosted:
```bash
oc get multiclusterhub -A
```

### 2. Delete the Conflicting Instance
Run a targeted deletion using the specific namespace uncovered in the previous step:
```bash
oc delete multiclusterhub multiclusterhub -n <target-namespace>
```

### 3. Clear Hidden Finalizers on Conflict
If the conflict deletion hangs, forcefully strip its finalizers to make room for your new deployment:
```bash
oc patch multiclusterhub multiclusterhub -n <target-namespace> --type=json -p '[{"op": "remove", "path": "/metadata/finalizers"}]'
```
Once `oc get multiclusterhub -A` returns no active objects, you can cleanly apply your new manifest configuration.

---

## Phase 5: Resolving Stuck Custom Resource Definitions (CRDs)
If creating a resource (like a `ClusterManagementAddOn` for Submariner) throws a forbidden error stating `create not allowed while custom resource definition is terminating`, the underlying CRD is stuck.

### 1. Locate the Terminating CRD
```bash
oc get crd | grep Terminating
```
*(Look for `clustermanagementaddons.addon.open-cluster-management.io`)*

### 2. Force Terminate the CRD by Clearing its Finalizers
```bash
oc patch crd clustermanagementaddons.addon.open-cluster-management.io --type=merge -p '{"metadata":{"finalizers":null}}'
```

### 3. Verify the CRD is Removed
```bash
oc get crd clustermanagementaddons.addon.open-cluster-management.io
```
*(It should return a `NotFound` error, enabling you or the operator to cleanly recreate it on the next deployment try).*


```bash
 oc get crd | grep -i open-cluster-management
backupschedules.cluster.open-cluster-management.io                         2026-09-05T21:52:03Z
channels.apps.open-cluster-management.io                                   2026-09-05T21:52:05Z
clusterinstances.siteconfig.open-cluster-management.io                     2026-09-05T21:52:08Z
clustermanagementaddons.addon.open-cluster-management.io                   2026-08-24T15:15:24Z
clustermanagers.operator.open-cluster-management.io                        2026-08-24T15:14:11Z
collectorconfigs.search.open-cluster-management.io                         2026-09-05T21:52:08Z
deployables.apps.open-cluster-management.io                                2026-09-05T21:52:05Z
gitopsclusters.apps.open-cluster-management.io                             2026-09-05T21:52:05Z
helmreleases.apps.open-cluster-management.io                               2026-09-05T21:52:05Z
internalhubcomponents.operator.open-cluster-management.io                  2026-09-05T21:52:05Z
klusterletaddonconfigs.agent.open-cluster-management.io                    2026-09-05T21:52:04Z
managedclusteraddons.addon.open-cluster-management.io                      2026-08-24T15:15:24Z
managedclusters.cluster.open-cluster-management.io                         2026-08-24T15:15:24Z
managedclustersets.cluster.open-cluster-management.io                      2026-08-24T15:15:24Z
manifestworks.work.open-cluster-management.io                              2026-08-24T15:15:24Z
multiclusterapplicationsetreports.apps.open-cluster-management.io          2026-09-05T21:52:05Z
multiclusterhubs.operator.open-cluster-management.io                       2026-09-05T21:26:59Z
multiclusterobservabilities.observability.open-cluster-management.io       2026-09-05T21:52:07Z
multiclusterroleassignments.rbac.open-cluster-management.io                2026-09-05T21:52:04Z
observabilityaddons.observability.open-cluster-management.io               2026-09-05T21:52:08Z
placementbindings.policy.open-cluster-management.io                        2026-09-05T21:52:04Z
placementrules.apps.open-cluster-management.io                             2026-09-05T21:52:05Z
policies.policy.open-cluster-management.io                                 2026-09-05T21:52:05Z
policyautomations.policy.open-cluster-management.io                        2026-09-05T21:52:05Z
policysets.policy.open-cluster-management.io                               2026-09-05T21:52:05Z
restores.cluster.open-cluster-management.io                                2026-09-05T21:52:03Z
searches.search.open-cluster-management.io                                 2026-09-05T21:52:08Z
submarinerconfigs.submarineraddon.open-cluster-management.io               2026-09-05T21:52:09Z
submarinerdiagnoseconfigs.submarineraddon.open-cluster-management.io       2026-09-05T21:52:09Z
subscriptionreports.apps.open-cluster-management.io                        2026-09-05T21:52:05Z
subscriptions.apps.open-cluster-management.io                              2026-09-05T21:52:06Z
subscriptionstatuses.apps.open-cluster-management.io                       2026-09-05T21:52:06Z
userpreferences.console.open-cluster-management.io                         2026-09-05T21:52:04Z
```

