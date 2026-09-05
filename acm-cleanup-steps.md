# Advanced Cluster Management (ACM) Cleanup Guide

This markdown file consolidates all the troubleshooting and forced deletion steps executed to clear the stuck `MulticlusterHub` resource and completely wipe out the `open-cluster-management` namespace.

---

## Phase 1: Resolving the Initial Webhook Blocker

When attempting to delete the `MulticlusterHub` resource, the process initially failed because a validating webhook configuration was active, but its backing service (`multiclusterhub-operator-webhook`) had already been deleted.

### 1. Delete the Mutating and Validating Webhook Configurations
Remove the cluster-wide admission webhooks so Kubernetes stops trying to validate API requests against a missing service endpoint:
```bash
oc delete validatingwebhookconfiguration multiclusterhub.validating-webhook.open-cluster-management.io
oc delete mutatingwebhookconfiguration multiclusterhub.mutating-webhook.open-cluster-management.io
```

---

## Phase 2: Clearing Stuck Resource Finalizers

If the `MulticlusterHub` resource is stuck in an `Uninstalling` phase with the finalizer `finalizer.operator.open-cluster-management.io`, you must manually strip its finalizer block.

### Method A: Clear Finalizers via JSON Patch (Recommended)
Apply a strict JSON patch to explicitly remove the metadata finalizers array:
```bash
oc patch multiclusterhub multiclusterhub -n open-cluster-management --type=json -p '[{"op": "remove", "path": "/metadata/finalizers"}]'
```

### Method B: Edit the Live Object Manually
If the patch command fails or errors out, edit the object configuration live:
```bash
oc edit multiclusterhub multiclusterhub -n open-cluster-management
```
Scroll to the `metadata:` section, locate the `finalizers:` block, completely delete the array elements, and save the file.

---

## Phase 3: Purging the `open-cluster-management` Namespace

Once individual custom resources are cleared, proceed with wiping out the entire namespace and any remaining hidden dependencies.

### 1. Trigger Standard Namespace Deletion
```bash
oc delete namespace open-cluster-management
```

### 2. Sweep for Duplicate or Labeled Admission Webhooks
If the namespace remains stuck in `Terminating`, check for and delete alternative webhook configs that might match the deployment label:
```bash
oc delete mutatingwebhookconfiguration -l app=multiclusterhub-operator
oc delete validatingwebhookconfiguration -l app=multiclusterhub-operator
```

### 3. Check for and Remove Broken Custom APIServices
Advanced Cluster Management registers specialized cluster-wide API groups. If their backing pods are gone, the entire control loop hangs. 
Identify broken services:
```bash
oc get apiservice | grep False
```
If any returned items are related to `open-cluster-management`, remove them:
```bash
oc delete apiservice <broken-apiservice-name>
```

---

## Phase 4: The Ultimate Escape Hatch (Forced Namespace Wipe)

If the namespace remains locked in a cached `Terminating` loop due to internal cluster constraints, you can completely bypass the standard control plane lifecycle loops by directly clearing the namespace's finalizers.

### Remove Namespace Finalizers via Cluster API Raw Endpoint
```bash
# 1. Export the live namespace JSON data and drop the finalizer array using jq
oc get namespace open-cluster-management -o json | jq '.spec.finalizers = []' > ns-clean.json

# 2. Alternatively, manually edit ns-clean.json and change "finalizers": [ "kubernetes" ] to "finalizers": []

# 3. Post the cleaned payload directly to the cluster API raw finalize endpoint
oc replace --raw "/api/v1/namespaces/open-cluster-management/finalize" -f ns-clean.json
```
