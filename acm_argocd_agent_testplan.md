# RHACM Argo CD Agent Integration — Test Plan & Results

**Component:** GitOps Add-on with Argo CD Agent (Managed Mode)
**RHACM Version:** 2.16
**Date:** 2026-05-21
**References:**
- [Validation & Test Engineering Report (PDF)](https://github.com/ch-stark/acmargocdagenttesting/blob/main/argocd_agent_test_report.pdf)
- [GitOps Documentation Rewrite](https://github.com/mikeshng/rhacm-docs/blob/gitops-rewrite/gitops/gitops-documentation-rewrite.md)

---

## 1. Test Environment & Infrastructure

| Item | Details |
|------|---------|
| Hub cluster | RHACM hub with OpenShift GitOps operator installed |
| Managed clusters | OCP cluster(s), Kind cluster (non-OCP) |
| Automation | E2E test automation driven by PICS team (weekly), targeting OCP managed clusters |
| Non-OCP coverage | Manual execution with extra setup when specific features require non-OCP validation |
| Argo CD Agent modes | Managed mode (agent pull model) |

> **Note:** PICS automation currently tests primarily against one OCP managed cluster per environment. Non-OCP cluster testing (Kind, MicroShift, etc.) is performed manually as needed.

---

## 2. Pre-Requisites (Common Setup)

Before executing any test case, the following must be in place:

1. **Create GitOpsCluster** resource with `gitopsAddon.enabled: true` and `argoCDAgent.enabled: true`
2. For OCP managed clusters, set `olmSubscription.enabled: true` in `gitopsAddon` spec if OLM auto-detection is needed
3. **Ensure `gitops-addon` ManagedClusterAddOn** exists on both the **local-cluster** and the **OCP managed cluster**
4. **Wait for the three add-on pods** in `openshift-gitops` namespace on the managed cluster:
   - `gitops-agent` (or `argocd-agent`)
   - `gitops-addon-agent`
   - `gitops-addon-agent-workmanager` (or equivalent controller pod)
5. Verify pods reach `Running` state and `Ready` condition before proceeding

```bash
# Verify ManagedClusterAddOn
oc get managedclusteraddons -n <managed-cluster-name> gitops-addon

# Verify pods on managed cluster
oc get pods -n openshift-gitops --context <managed-cluster-context>
```

---

## 3. Test Cases & Results

### 3.0 — Setup & Lifecycle Tests

#### TC-SETUP-01: Create GitOpsCluster with OLM Enabled

| Field | Value |
|-------|-------|
| **Objective** | Create a `GitOpsCluster` resource with `olmSubscription.enabled: true`, verify the gitops-addon MCA is created on local-cluster and OCP managed cluster, and confirm three addon pods start in `openshift-gitops` |
| **Pre-condition** | Hub cluster running, managed cluster(s) registered, Placement targeting managed clusters exists |
| **Steps** | 1. Apply `GitOpsCluster` with `gitopsAddon.enabled: true` and `olmSubscription.enabled: true`<br>2. Check `managedclusteraddon/gitops-addon` on local-cluster<br>3. Check `managedclusteraddon/gitops-addon` on managed OCP cluster<br>4. Wait for 3 addon pods in `openshift-gitops` on managed cluster |
| **Expected** | MCA created on both clusters; all three pods `Running` and `Ready` within 5 minutes |
| **Result** | **PASSED** |
| **Evidence** | MCA present on local-cluster and managed cluster; pods confirmed running |

```yaml
apiVersion: apps.open-cluster-management.io/v1beta1
kind: GitOpsCluster
metadata:
  name: agent-gitops
  namespace: openshift-gitops
spec:
  argoServer:
    cluster: local-cluster
  argoNamespace: openshift-gitops
  placementRef:
    kind: Placement
    apiVersion: cluster.open-cluster-management.io/v1beta1
    name: agent-placement
  gitopsAddon:
    enabled: true
    olmSubscription:
      enabled: true
    argoCDAgent:
      enabled: true
      mode: managed
```

---

#### TC-SETUP-02: Uninstall / Reinstall GitOps Addon — Enable Agent Mode

| Field | Value |
|-------|-------|
| **Objective** | Uninstall gitops-addon, then reinstall with agent mode enabled; verify clean transition |
| **Pre-condition** | GitOps addon previously installed on managed cluster |
| **Steps** | 1. Delete `GitOpsCluster` resource<br>2. Verify MCA is removed and addon pods are terminated on managed cluster<br>3. Verify spoke resources cleaned up (no orphaned finalizers)<br>4. Re-create `GitOpsCluster` with `argoCDAgent.enabled: true, mode: managed`<br>5. Confirm addon pods restart in agent mode |
| **Expected** | Clean uninstall with no orphaned resources; reinstall brings up agent-mode pods without manual intervention |
| **Result** | **PASSED** |
| **Evidence** | Finalizer cascade completed cleanly (TC-07 from prior report); agent pods started after re-apply |

---

### 3.1 — Standalone ArgoCD Application Tests

#### TC-STANDALONE-01: Standalone ArgoCD App Reconciled on Managed Cluster

| Field | Value |
|-------|-------|
| **Objective** | Verify a standalone `Application` (created directly on managed cluster's ArgoCD) can reconcile without the agent/pull model |
| **Pre-condition** | OpenShift GitOps operator installed on managed cluster; ArgoCD instance running |
| **Steps** | 1. Create an `Application` resource directly on the managed cluster's ArgoCD in `openshift-gitops` namespace<br>2. Point to a known repo (e.g., `argoproj/argocd-example-apps`, path: `guestbook`)<br>3. Set `syncPolicy.automated` with `prune: true, selfHeal: true`<br>4. Wait for sync<br>5. Verify workload deployed in target namespace |
| **Expected** | Application syncs and becomes `Healthy`/`Synced`; guestbook pods running on managed cluster |
| **Result** | **PASSED** |
| **Evidence** | `oc get app guestbook -n openshift-gitops` shows Synced/Healthy; workload pods confirmed |

---

### 3.2 — Basic Pull Model Tests

#### TC-PULL-01: Basic Pull Model in `openshift-gitops` Namespace on Managed Cluster

| Field | Value |
|-------|-------|
| **Objective** | Verify the basic pull model works when Application is created on the hub and pulled to the managed cluster's `openshift-gitops` namespace |
| **Pre-condition** | GitOpsCluster with agent mode enabled; hub principal configured; companion policy with RBAC applied |
| **Steps** | 1. Create `Application` on hub ArgoCD targeting managed cluster<br>2. Set destination to managed cluster, namespace `guestbook`<br>3. Verify Application spec is pulled to managed cluster's `openshift-gitops` namespace<br>4. Verify workload is deployed on managed cluster<br>5. Verify status syncs back to hub (read-only mirror) |
| **Expected** | Application appears on managed cluster's ArgoCD; workload deployed; hub shows synced status |
| **Result** | **PASSED** |
| **Evidence** | Hub shows application status; managed cluster has running workload |

---

#### TC-PULL-02: Basic Pull Model in Any Namespace on Managed Cluster — Default AppProject

| Field | Value |
|-------|-------|
| **Objective** | Verify pull model works with Application deployed into a non-default namespace, using the `default` AppProject |
| **Pre-condition** | Agent mode enabled; `sourceNamespaces: ["*"]` configured on hub principal; RBAC policy applied |
| **Steps** | 1. Create `Application` on hub targeting managed cluster namespace `test-app-ns`<br>2. Use `project: default`<br>3. Ensure companion policy creates the target namespace on managed cluster<br>4. Verify application syncs and workload deploys in `test-app-ns`<br>5. Verify hub reflects status |
| **Expected** | Application synced in arbitrary namespace using default project; workload running |
| **Result** | **PASSED** |
| **Evidence** | Workload pods confirmed in `test-app-ns`; hub status mirror accurate |

---

#### TC-PULL-03: Basic Pull Model in Any Namespace on Managed Cluster — Custom AppProject

| Field | Value |
|-------|-------|
| **Objective** | Verify pull model works with a custom `AppProject` restricting source repos and destination namespaces |
| **Pre-condition** | Agent mode enabled; custom `AppProject` created on hub |
| **Steps** | 1. Create custom `AppProject` on hub with restricted `sourceRepos` and `destinations`<br>2. Create `Application` on hub referencing the custom project<br>3. Target a namespace allowed by the AppProject<br>4. Verify application syncs correctly on managed cluster<br>5. Verify that deploying outside the allowed destinations is rejected |
| **Expected** | Application syncs when within project boundaries; out-of-scope destinations are denied |
| **Result** | **PASSED** |
| **Evidence** | Custom project scoping enforced; workload deployed only in permitted namespace |

---

### 3.3 — Advanced (Agent) Pull Model Tests

#### TC-AGENT-01: Advanced Pull Model with ApplicationSet on Managed Cluster

| Field | Value |
|-------|-------|
| **Objective** | Verify `ApplicationSet` with `clusterDecisionResource` generator auto-targets agent-registered managed clusters |
| **Pre-condition** | Agent mode enabled; `PlacementBinding` and companion policy applied; `acm-placement` ConfigMap generated |
| **Steps** | 1. Apply `ApplicationSet` on hub using `clusterDecisionResource` generator referencing `acm-placement`<br>2. Use label selector matching `agent-placement`<br>3. Set `destination.name: '{{name}}'` for dynamic cluster targeting<br>4. Verify Application instances generated per managed cluster<br>5. Verify workload deployed on each targeted cluster<br>6. Verify hub shows all generated applications with correct status |
| **Expected** | One `Application` per matched cluster; all synced and healthy; workloads running |
| **Result** | **PASSED** |
| **Evidence** | `guestbook-<cluster>` apps created per cluster; workloads confirmed |

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook-agent-deployment
  namespace: openshift-gitops
spec:
  generators:
    - clusterDecisionResource:
        configMapRef: acm-placement
        labelSelector:
          matchLabels:
            cluster.open-cluster-management.io/placement: agent-placement
        requeueAfterSeconds: 180
  template:
    metadata:
      name: 'guestbook-{{name}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps
        targetRevision: HEAD
        path: guestbook
      destination:
        name: '{{name}}'
        namespace: guestbook
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

---

### 3.4 — Local-Cluster Support Tests

#### TC-LOCAL-01: GitOps Addon Local-Cluster Support — Agent Mode

| Field | Value |
|-------|-------|
| **Objective** | Verify gitops-addon works on hub's `local-cluster` in agent mode; all pods running including agent |
| **Pre-condition** | GitOpsCluster placement includes `local-cluster`; agent mode enabled |
| **Steps** | 1. Ensure Placement selects `local-cluster`<br>2. Apply `GitOpsCluster` with `argoCDAgent.enabled: true, mode: managed`<br>3. Verify `managedclusteraddon/gitops-addon` created for `local-cluster`<br>4. Verify all addon pods (including agent pod) running in hub's `openshift-gitops` namespace<br>5. Verify agent establishes mTLS connection to principal (loopback on hub)<br>6. Deploy a test application and verify pull model sync on local-cluster |
| **Expected** | Agent pods running; mTLS handshake successful; application syncs via pull model on local-cluster |
| **Result** | **PASSED** |
| **Evidence** | Agent pod logs show successful principal connection; workload deployed |

---

## 4. Prior Validation Matrix (from PDF Report)

The following results are carried forward from the [formal validation report](https://github.com/ch-stark/acmargocdagenttesting/blob/main/argocd_agent_test_report.pdf) (8/8 scenarios passed, 2 target environments, 100% manifest compliance):

| Test ID | Scope / Context | Assertion / Verification Condition | Status |
|---------|----------------|-------------------------------------|--------|
| TC-01 | Hub Principal | Validate local controller deactivation and mTLS listener initialization patterns via specific logs | **PASSED** — mTLS handshakes verified on Principal logs |
| TC-02 | Hub Control | Confirm `managedclusteraddon` generation and check cluster proxy secret generation mapping parameters | **PASSED** — Secret endpoints populate mapping structures successfully |
| TC-03 | OCP Spoke | Verify OLM auto-detection loop configures `DISABLE_DEFAULT_ARGOCD_INSTANCE=true` systematically | **PASSED** — Subscription env validation verified |
| TC-04 | OCP Workload | Audit deployment placement sync. Ensure expected container failure occurs due to OpenShift SCC restricted-v2 security boundaries (Port 80 binding lock) | **EXPECTED FAILURE** — Deployment successfully synced; SCC block active as intended |
| TC-05 | Kind Spoke | Verify non-OCP context safety fallback. Confirm OLM subscription skipped and native chart templates applied | **PASSED** — Deployment completely functional on localized container topology |
| TC-06 | Kind Images | Audit image source declarations. Verify standard Red Hat enterprise registry fallback to external public nodes | **PASSED** — Image references resolved cleanly without pull timeouts |
| TC-07 | Teardown Lifecycle | Execute structured cascading finalizer validation (ApplicationSet -> GitOpsCluster -> Policies -> Placements) | **PASSED** — Zero finalizer hangs encountered; zero manual intervention required |
| TC-08 | Spoke Post-Cleanup | Ensure target namespaces and customer user resources stay isolated on spokes while infrastructure add-ons are swept away | **VERIFIED** — User data preserved; core system operators removed clean |

---

## 5. Test Automation & Coverage Notes

### Current Automation Coverage

- **PICS e2e automation** runs weekly across hub + OCP managed clusters
- ALC automation primarily targets **one OCP managed cluster** per test environment
- Hub is set up by PICS with different kinds of managed clusters (most are OCP)

### Manual Testing Scope

Since automation does not fully support non-OCP clusters, the following are tested manually with extra setup:

- Kind cluster as managed cluster (TC-05, TC-06 from prior report)
- MicroShift or other non-OCP distributions when specific features require it
- Agent version drift heal validation

### Gaps / Future Automation Candidates

| Area | Current State | Target |
|------|--------------|--------|
| Non-OCP managed clusters | Manual | Automate Kind cluster in CI |
| Agent mode install/uninstall cycling | Manual | Automate lifecycle transitions |
| Custom AppProject enforcement | Manual | Add negative test cases to automation |
| Multi-cluster ApplicationSet scaling | Not tested | Add 3+ cluster topology tests |

---

## 6. Test Results Summary

| Category | Total | Passed | Expected Failure | Pending |
|----------|-------|--------|-----------------|---------|
| Setup & Lifecycle (TC-SETUP) | 2 | 2 | 0 | 0 |
| Standalone ArgoCD (TC-STANDALONE) | 1 | 1 | 0 | 0 |
| Basic Pull Model (TC-PULL) | 3 | 3 | 0 | 0 |
| Advanced Agent Pull Model (TC-AGENT) | 1 | 1 | 0 | 0 |
| Local-Cluster Support (TC-LOCAL) | 1 | 1 | 0 | 0 |
| Prior Validation Matrix (TC-01..08) | 8 | 7 | 1 | 0 |
| **Total** | **16** | **15** | **1** | **0** |

**Overall Status: ALL TESTS PASSING** (TC-04 expected failure by design — SCC restricted-v2 boundary enforcement)

---

## 7. Hub Cluster Principal Configuration Reference

```bash
export KUBECONFIG=/path/to/hub/kubeconfig

kubectl patch argocd openshift-gitops -n openshift-gitops --type=merge -p '{
  "spec": {
    "controller": { "enabled": false },
    "argoCDAgent": {
      "principal": {
        "enabled": true,
        "auth": "mtls:CN=system:open-cluster-management:cluster:([^:]+):addon:gitops-addon:agent:gitops-addon-agent",
        "namespace": { "allowedNamespaces": ["*"] },
        "server": { "route": { "enabled": true } }
      }
    },
    "sourceNamespaces": ["*"]
  }
}'
```

## 8. Teardown / Cleanup Sequence

```bash
# Step 1: Remove workload application objects first to allow clean un-sync
kubectl delete applicationset guestbook-agent-deployment -n openshift-gitops

# Step 2: Remove central GitOps manager configuration
kubectl delete gitopscluster agent-gitops -n openshift-gitops

# Step 3: Tear down custom infrastructure policies and tracking markers
kubectl delete policy agent-custom-spoke-setup -n openshift-gitops
kubectl delete placementbinding agent-custom-setup-binding -n openshift-gitops

# Step 4: Revoke base selection placements
kubectl delete placement agent-placement -n openshift-gitops
```
