# Test Plan Critical Review & Additional Test Cases

**Reviewing:** `acm_argocd_agent_testplan.md`
**Date:** 2026-05-21

---

## 1. Critical Findings

### 1.1 — Default `AppProject` Used Almost Everywhere

TC-PULL-01, TC-PULL-02, TC-AGENT-01, and TC-STANDALONE-01 all use `project: default`. The `default` AppProject permits:
- All source repos (`*`)
- All destination namespaces (`*`)
- All resource kinds (cluster and namespace-scoped)

TC-PULL-03 claims to test a custom AppProject but lacks:
- Specific YAML for the custom AppProject definition
- Verification of `clusterResourceWhitelist` / `clusterResourceBlacklist` enforcement
- Verification of `namespaceResourceBlacklist` (e.g., deny `Secret` creation via Git)
- Testing `sourceRepos` restrictions with the agent pull model specifically

**Risk:** Customers using custom AppProjects with restricted permissions will hit agent-specific edge cases (e.g., the agent may need permissions that the AppProject denies) that we've never tested.

---

### 1.2 — All Sync Policies Use `prune: true` — No Create/Update-Only Scenario

Every single test case sets:
```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

**What's missing:**
- No test with `prune: false` — user wants Git to create and update resources but **never delete** them
- No test with `selfHeal: false` — user wants manual control over drift correction
- No test with **no `automated` block at all** — manual sync only, user explicitly triggers syncs
- No test with the `argocd.argoproj.io/sync-options: Prune=false` annotation on individual resources
- No test for `Delete` propagation policy (`foreground` vs `background` vs `orphan`) on the managed cluster via the agent

**Risk:** This is a common production pattern. Many teams use GitOps for deployment but forbid Git-driven deletion to prevent accidental resource removal. If the agent handles prune differently than a direct ArgoCD controller, it's invisible in current tests.

---

### 1.3 — Policy Configuration Not Tested Beyond the Default

Based on the [multicloud-integrations source](https://github.com/stolostron/multicloud-integrations), the ArgoCD Policy is **user-owned** and configurable. The GitOpsCluster controller auto-generates a bare-minimum Policy containing only the ArgoCD CR — users are expected to customize it by adding RBAC, namespaces, Applications, and ArgoCD settings. The controller preserves user customizations and only touches `argoCDAgent.agent.image` during drift heal.

**Current test plan gap:** Every test case assumes a single Policy modification pattern (add `cluster-admin` ClusterRoleBinding + guestbook namespace). No test validates:

- **Policy customization survival** — does the controller actually preserve user-added templates after a reconcile loop?
- **Policy recreation behavior** — what happens when the Policy is deleted? (Controller recreates unless `skip-argocd-policy` annotation is set)
- **`skip-argocd-policy` annotation** — verified by e2e but not in our integration test plan
- **Agent mode Policy differences** — in agent mode, Applications are NOT added to the Policy (they're dispatched by the principal); in autonomous mode, the `default` AppProject IS auto-included
- **Adding `view` ClusterRoleBinding** for the agent SA to the Policy — required for ArgoCD UI live manifest but never tested
- **`destinationBasedMapping` consistency** — principal and agent must match or Redis keys diverge and UI breaks

**Key design principle from the source code:**

| Policy aspect | Controller behavior | User responsibility |
|--------------|-------------------|-------------------|
| ArgoCD CR | Auto-generated | Can customize ArgoCD settings in the template |
| RBAC (ClusterRoleBinding) | **Not included** | Must add `cluster-admin` (or custom) binding |
| Namespaces | **Not included** | Must add target namespaces (or use `CreateNamespace=true`) |
| Applications (non-agent) | **Not included** | Can add directly to Policy |
| Applications (agent mode) | **Not in Policy** | Created on hub, dispatched by principal |
| `default` AppProject (autonomous) | Auto-included | N/A — controller adds it automatically |
| `argoCDAgent.agent.image` | Patched during drift heal | Can disable via `skip-agent-version-heal` annotation |
| All other customizations | **Preserved across reconciles** | User owns and maintains |

**Critical prerequisites NOT in the Policy that are easy to miss:**

| Prerequisite | Where to configure | What breaks without it |
|-------------|-------------------|----------------------|
| `ManagedClusterSetBinding` for `default` in `openshift-gitops` | Hub, manually | Placement finds zero clusters |
| `default` AppProject wildcards (`sourceNamespaces: ["*"]`, `destinations: [{name:"*", namespace:"*", server:"*"}]`, `clusterResourceWhitelist: [{group:"*", kind:"*"}]`, `sourceRepos: ["*"]`) | Hub ArgoCD, manually | Principal skips AppProject propagation; agents fail with "project not found" |
| `ARGOCD_CLUSTER_CONFIG_NAMESPACES=openshift-gitops,local-cluster` | OLM Subscription env var | Agent/principal pods crash with "namespaces is forbidden" |
| `view` ClusterRoleBinding for agent SA (`acm-openshift-gitops-agent-agent`) | Managed cluster, manually or via Policy | ArgoCD UI live manifest returns "Resource not found in cluster" |
| `destinationBasedMapping: true` on agent ArgoCD CR | Policy object template | Redis key mismatch between principal/agent; UI live manifest breaks |

**Risk:** Users who follow the documentation step-by-step will miss these prerequisites because they're scattered across different configuration layers. The test plan should validate that missing any single prerequisite produces a clear error, not a silent failure.

---

### 1.4 — Other Gaps

| Gap | Impact |
|-----|--------|
| **No `allowedNamespaces` restriction test** — hub principal is configured with `allowedNamespaces: ["*"]` in every test | Unknown whether namespace restrictions on the principal are enforced through the agent |
| **No agent version drift heal test** — the doc describes auto-patching agent images when the principal upgrades, but no test validates it | Upgrade scenarios untested |
| **No network disruption / reconnection test** — what happens when the agent loses mTLS connection to the principal and reconnects? | Resilience unknown |
| **No multi-AppProject test** — what if two Applications in different AppProjects target the same managed cluster? | Project isolation on the agent side untested |
| **No `syncPolicy.syncOptions` coverage** — `CreateNamespace`, `ServerSideApply`, `FailOnSharedResource` | Agent may behave differently than direct controller for these options |
| **No PlacementDecision `score` workaround test** — OCM adds `score:0` (int64) to PlacementDecision; ArgoCD's DuckType generator panics on non-string values | ApplicationSet with `clusterDecisionResource` will crash in production without the manual PlacementDecision workaround |
| **No cert rotation test** — `argocd-agent-client-tls` has 24-hour validity; OCM rotates at ~80% lifetime | Unknown whether agent survives cert rotation without disruption |

---

## 2. Additional Test Cases

### 2.1 — AppProject Enforcement Tests

#### TC-PROJECT-01: AppProject with Restricted `sourceRepos`

| Field | Value |
|-------|-------|
| **Objective** | Verify the agent enforces `sourceRepos` restrictions — only whitelisted Git repos can be synced |
| **Pre-condition** | Agent mode enabled |
| **Steps** | 1. Create AppProject `restricted-sources` with `sourceRepos: ["https://github.com/argoproj/argocd-example-apps"]`<br>2. Create Application referencing this project, using the allowed repo<br>3. Verify sync succeeds<br>4. Create a second Application referencing this project but pointing to `https://github.com/some-other-org/some-repo`<br>5. Verify the second application is rejected |
| **Expected** | Allowed repo syncs; disallowed repo is denied by AppProject policy |
| **Result** | **NOT YET TESTED** |

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: restricted-sources
  namespace: openshift-gitops
spec:
  description: "Only allow specific source repos"
  sourceRepos:
    - "https://github.com/argoproj/argocd-example-apps"
  destinations:
    - namespace: "guestbook"
      server: "*"
  clusterResourceWhitelist: []
  namespaceResourceWhitelist:
    - group: ""
      kind: "*"
    - group: "apps"
      kind: "*"
```

---

#### TC-PROJECT-02: AppProject with `namespaceResourceBlacklist` — Deny Secrets via Git

| Field | Value |
|-------|-------|
| **Objective** | Verify the agent enforces `namespaceResourceBlacklist` to prevent specific resource kinds (e.g., `Secret`) from being managed via GitOps |
| **Pre-condition** | Agent mode enabled |
| **Steps** | 1. Create AppProject `no-secrets` with `namespaceResourceBlacklist: [{group: "", kind: "Secret"}]`<br>2. Create Application whose Git source includes a `Deployment` and a `Secret`<br>3. Verify the `Deployment` syncs<br>4. Verify the `Secret` is rejected by the AppProject policy<br>5. Verify hub reflects the partial sync / error |
| **Expected** | Deployment created; Secret creation blocked by AppProject; Application shows sync error for the Secret resource |
| **Result** | **NOT YET TESTED** |

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: no-secrets
  namespace: openshift-gitops
spec:
  sourceRepos: ["*"]
  destinations:
    - namespace: "*"
      server: "*"
  namespaceResourceBlacklist:
    - group: ""
      kind: "Secret"
  clusterResourceWhitelist: []
```

---

#### TC-PROJECT-03: AppProject Denying Cluster-Scoped Resources

| Field | Value |
|-------|-------|
| **Objective** | Verify an AppProject with empty `clusterResourceWhitelist` prevents creation of cluster-scoped resources (Namespaces, ClusterRoles, ClusterRoleBindings) via the agent |
| **Pre-condition** | Agent mode enabled |
| **Steps** | 1. Create AppProject with `clusterResourceWhitelist: []`<br>2. Create Application whose Git source includes a `ClusterRole` and a `Namespace`<br>3. Verify both cluster-scoped resources are denied<br>4. Verify namespace-scoped resources in the same app still sync |
| **Expected** | Cluster-scoped resources blocked; namespace-scoped resources permitted |
| **Result** | **NOT YET TESTED** |

---

#### TC-PROJECT-04: Multiple AppProjects on Same Managed Cluster

| Field | Value |
|-------|-------|
| **Objective** | Verify two Applications using different AppProjects can target the same managed cluster without cross-project interference |
| **Pre-condition** | Agent mode enabled |
| **Steps** | 1. Create AppProject `team-a` allowing destination namespace `team-a-ns`<br>2. Create AppProject `team-b` allowing destination namespace `team-b-ns`<br>3. Deploy Application in `team-a` project to `team-a-ns`<br>4. Deploy Application in `team-b` project to `team-b-ns`<br>5. Verify both sync independently<br>6. Attempt to deploy `team-a` Application to `team-b-ns` — verify rejection<br>7. Verify no cross-contamination of resources |
| **Expected** | Both applications sync in their own namespaces; cross-namespace deployment denied by AppProject |
| **Result** | **NOT YET TESTED** |

---

### 2.2 — ApplicationSet Generator & Resilience Tests

The current test plan only covers `clusterDecisionResource` generator (TC-AGENT-01, TC-SYNC-07). The following generators and resilience scenarios are untested.

#### TC-APPSET-01: GitGenerator — Directory-Based ApplicationSet via Agent

| Field | Value |
|-------|-------|
| **Objective** | Verify an ApplicationSet using `git.directories` generator creates one Application per directory in a Git repo, synced to managed clusters via the agent pull model |
| **Pre-condition** | Agent mode enabled; Git repo with multiple app directories (e.g., `apps/app-a/`, `apps/app-b/`, `apps/app-c/`) |
| **Steps** | 1. Create ApplicationSet with `git` generator scanning directories under `apps/`<br>2. Set destination to a specific managed cluster via `destination.name`<br>3. Verify one Application is generated per directory<br>4. Verify each Application syncs and deploys its workload on the managed cluster<br>5. **Test add:** Add a new directory `apps/app-d/` to Git — verify a new Application is auto-generated and synced<br>6. **Test remove:** Remove directory `apps/app-c/` from Git — verify the Application is removed (or preserved per `preserveResourcesOnDeletion`)<br>7. Verify hub mirrors all generated Applications and their statuses |
| **Expected** | One Application per Git directory; dynamic addition/removal works through the agent |
| **Result** | **NOT YET TESTED** |

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: git-directory-apps
  namespace: openshift-gitops
spec:
  generators:
    - git:
        repoURL: https://github.com/argoproj/argocd-example-apps
        revision: HEAD
        directories:
          - path: "apps/*"
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        name: ocp-cluster1
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: false
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

---

#### TC-APPSET-02: GitGenerator — File-Based ApplicationSet via Agent

| Field | Value |
|-------|-------|
| **Objective** | Verify an ApplicationSet using `git.files` generator reads JSON/YAML config files from Git to parameterize Applications deployed through the agent |
| **Pre-condition** | Agent mode enabled; Git repo with config files (e.g., `config/cluster-a.json`, `config/cluster-b.json`) containing per-cluster parameters |
| **Steps** | 1. Create config files in Git with per-cluster parameters (namespace, replicas, etc.)<br>2. Create ApplicationSet with `git.files` generator pointing to `config/*.json`<br>3. Verify Applications are generated with correct per-cluster parameters<br>4. Verify workloads deploy with the specified parameters on managed clusters<br>5. **Test update:** Modify a config file (e.g., change replicas) — verify the Application and workload update accordingly<br>6. Verify hub reflects all generated Applications |
| **Expected** | Applications parameterized from Git config files; updates to config files propagate through the agent |
| **Result** | **NOT YET TESTED** |

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: git-file-apps
  namespace: openshift-gitops
spec:
  generators:
    - git:
        repoURL: https://github.com/your-org/gitops-config
        revision: HEAD
        files:
          - path: "config/*.json"
  template:
    metadata:
      name: 'app-{{cluster.name}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps
        targetRevision: HEAD
        path: guestbook
      destination:
        name: '{{cluster.name}}'
        namespace: '{{app.namespace}}'
      syncPolicy:
        automated:
          prune: false
          selfHeal: true
```

---

#### TC-APPSET-03: MatrixGenerator — ClusterDecision + GitGenerator Combined

| Field | Value |
|-------|-------|
| **Objective** | Verify a Matrix generator combining `clusterDecisionResource` (which clusters) with `git.directories` (which apps) produces the correct cross-product of Applications through the agent |
| **Pre-condition** | Agent mode enabled; Placement selecting 2+ clusters; Git repo with 2+ app directories |
| **Steps** | 1. Create ApplicationSet with `matrix` generator combining `clusterDecisionResource` and `git.directories`<br>2. Placement selects clusters `cluster-a` and `cluster-b`; Git has directories `apps/frontend/` and `apps/backend/`<br>3. Verify 4 Applications are generated (2 clusters x 2 apps): `frontend-cluster-a`, `frontend-cluster-b`, `backend-cluster-a`, `backend-cluster-b`<br>4. Verify all 4 sync and deploy correct workloads to the correct clusters and namespaces<br>5. **Test add cluster:** Add `cluster-c` to Placement — verify 2 new Applications generated (`frontend-cluster-c`, `backend-cluster-c`)<br>6. **Test add app:** Add `apps/api/` to Git — verify 3 new Applications generated (one per cluster)<br>7. **Test remove cluster:** Remove `cluster-b` from Placement — verify 2 Applications removed (or resources preserved per `preserveResourcesOnDeletion`)<br>8. Verify hub mirrors the full matrix of Applications and their statuses |
| **Expected** | Cross-product of clusters x apps; dynamic addition/removal of clusters and apps updates the matrix correctly |
| **Result** | **NOT YET TESTED** |

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: matrix-cluster-apps
  namespace: openshift-gitops
spec:
  generators:
    - matrix:
        generators:
          - clusterDecisionResource:
              configMapRef: acm-placement
              labelSelector:
                matchLabels:
                  cluster.open-cluster-management.io/placement: agent-placement
              requeueAfterSeconds: 180
          - git:
              repoURL: https://github.com/argoproj/argocd-example-apps
              revision: HEAD
              directories:
                - path: "apps/*"
  syncPolicy:
    preserveResourcesOnDeletion: true
  template:
    metadata:
      name: '{{path.basename}}-{{name}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        name: '{{name}}'
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: false
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

---

#### TC-APPSET-04: Managed Cluster Disconnected — Agent Loses Connection to Principal

| Field | Value |
|-------|-------|
| **Objective** | Verify behavior when a managed cluster loses connectivity to the hub (agent cannot reach the principal): workloads continue running, status goes stale on hub, and reconnection re-syncs cleanly |
| **Pre-condition** | Agent mode enabled; Applications synced and healthy on managed cluster |
| **Steps** | 1. Verify Application is `Synced`/`Healthy` on both hub and managed cluster<br>2. **Simulate disconnect:** Block network connectivity between the managed cluster and the hub (e.g., firewall rule blocking the principal route, or scale down the cluster-proxy)<br>3. Verify workloads **continue running** on the managed cluster — the agent operates independently<br>4. Verify hub shows Application status becomes stale / unknown / loses heartbeat<br>5. **Test drift during disconnect:** Make a change in Git while disconnected — verify the managed cluster does NOT sync (agent cannot reach principal for new specs)<br>6. **Restore connectivity:** Remove the network block<br>7. Verify agent reconnects to principal via mTLS<br>8. Verify pending Git changes now sync to managed cluster<br>9. Verify hub status updates to reflect current state<br>10. Verify no duplicate resources or conflicts from the reconnection |
| **Expected** | Workloads survive disconnection; hub status goes stale; reconnection triggers clean re-sync without duplication |
| **Result** | **NOT YET TESTED** |

---

#### TC-APPSET-05: Managed Cluster Updated (OCP Upgrade) — Agent Continuity

| Field | Value |
|-------|-------|
| **Objective** | Verify the agent continues functioning correctly after a managed cluster OCP upgrade, and the agent version drift heal mechanism patches the agent image if needed |
| **Pre-condition** | Agent mode enabled; Applications synced and healthy; managed cluster on OCP version N |
| **Steps** | 1. Record current agent pod image version on managed cluster<br>2. Record current principal Deployment image version on hub<br>3. **Upgrade managed cluster** from OCP version N to N+1 (or trigger an OpenShift GitOps operator upgrade on the hub)<br>4. Verify addon pods restart successfully after upgrade<br>5. Verify agent re-establishes mTLS connection to principal<br>6. Verify existing Applications remain `Synced`/`Healthy`<br>7. **Test version drift heal:** If the hub's OpenShift GitOps operator is upgraded (new principal image), verify the hub controller auto-patches the ArgoCD Policy to enforce the updated agent image on managed clusters<br>8. Verify the managed cluster agent pod image is updated to match the principal<br>9. Verify no Application disruption during the agent image rollout<br>10. Verify `apps.open-cluster-management.io/skip-agent-version-heal: "true"` annotation disables the auto-patch when set |
| **Expected** | Agent survives cluster upgrade; version drift heal patches agent image automatically; Applications remain healthy throughout |
| **Result** | **NOT YET TESTED** |

---

#### TC-APPSET-06: Managed Cluster Disconnected with ApplicationSet — Resource Preservation

| Field | Value |
|-------|-------|
| **Objective** | Verify that when a managed cluster is disconnected AND removed from the Placement, spoke resources are preserved (with `preserveResourcesOnDeletion: true`), and re-adding the cluster after reconnection reconciles cleanly |
| **Pre-condition** | Agent mode enabled; ApplicationSet with `preserveResourcesOnDeletion: true` and `prune: false`; workloads deployed on managed cluster |
| **Steps** | 1. Verify ApplicationSet has generated Applications and workloads are running on managed cluster<br>2. **Simulate disconnect:** Block network from managed cluster to hub<br>3. **Remove cluster from Placement** while disconnected (change Placement label selector on hub)<br>4. Verify the generated Application is removed from the hub<br>5. Verify the hub cannot confirm spoke resource state (cluster is disconnected)<br>6. **Restore connectivity:** Remove the network block<br>7. Verify workloads are **still running** on the managed cluster (not deleted — `preserveResourcesOnDeletion` + disconnect means no cascade)<br>8. **Re-add cluster to Placement:** Restore the label selector<br>9. Verify ApplicationSet regenerates the Application<br>10. Verify agent re-syncs — existing resources are adopted, not duplicated<br>11. Verify hub shows `Synced`/`Healthy` again |
| **Expected** | Resources survive both disconnection and Placement removal; reconnection + re-add results in clean adoption without duplication |
| **Result** | **NOT YET TESTED** |

---

### 2.3 — Sync Policy / Prune Protection Tests

#### TC-SYNC-01: Create and Update Only — `prune: false`

| Field | Value |
|-------|-------|
| **Objective** | Verify that with `prune: false`, removing a resource from Git does NOT delete it from the managed cluster |
| **Pre-condition** | Agent mode enabled |
| **Steps** | 1. Create Application with `syncPolicy.automated.prune: false, selfHeal: true`<br>2. Git source contains `Deployment` + `Service` + `ConfigMap`<br>3. Wait for initial sync — all 3 resources created on managed cluster<br>4. Remove the `ConfigMap` from the Git source<br>5. Wait for ArgoCD to detect drift<br>6. Verify the `ConfigMap` is NOT deleted from the managed cluster<br>7. Verify Application shows `OutOfSync` (resource exists on cluster but not in Git)<br>8. Verify hub mirrors the `OutOfSync` state |
| **Expected** | ConfigMap remains on managed cluster; Application reports OutOfSync; no deletion occurs |
| **Result** | **NOT YET TESTED** |

```yaml
syncPolicy:
  automated:
    prune: false
    selfHeal: true
```

---

#### TC-SYNC-02: Create and Update Only — No Automated Sync (Manual Only)

| Field | Value |
|-------|-------|
| **Objective** | Verify that without an `automated` sync policy, the agent does not auto-sync and the user must trigger syncs manually |
| **Pre-condition** | Agent mode enabled |
| **Steps** | 1. Create Application with NO `syncPolicy.automated` block<br>2. Verify Application appears on managed cluster as `OutOfSync`<br>3. Verify no resources are deployed automatically<br>4. Manually trigger sync via `argocd app sync` or UI<br>5. Verify resources are created after manual sync<br>6. Modify a resource in Git<br>7. Verify Application shows `OutOfSync` but does NOT auto-reconcile<br>8. Verify hub accurately reflects the OutOfSync state |
| **Expected** | No automatic sync; manual sync deploys resources; subsequent changes require manual sync |
| **Result** | **NOT YET TESTED** |

```yaml
spec:
  syncPolicy: {}
```

---

#### TC-SYNC-03: Per-Resource Prune Protection via Annotation

| Field | Value |
|-------|-------|
| **Objective** | Verify that individual resources annotated with `argocd.argoproj.io/sync-options: Prune=false` are protected from deletion even when the Application has `prune: true` |
| **Pre-condition** | Agent mode enabled |
| **Steps** | 1. Create Application with `prune: true`<br>2. Git source contains a `Deployment` (no annotation) and a `ConfigMap` with annotation `argocd.argoproj.io/sync-options: Prune=false`<br>3. Wait for initial sync<br>4. Remove BOTH resources from Git source<br>5. Wait for sync<br>6. Verify the `Deployment` IS deleted (prune works)<br>7. Verify the `ConfigMap` is NOT deleted (annotation protects it)<br>8. Verify hub shows correct status for each resource |
| **Expected** | Deployment pruned; ConfigMap retained due to annotation |
| **Result** | **NOT YET TESTED** |

---

#### TC-SYNC-04: `selfHeal: false` — Allow Manual Cluster-Side Changes

| Field | Value |
|-------|-------|
| **Objective** | Verify that with `selfHeal: false`, manual changes made directly on the managed cluster are NOT reverted by the agent |
| **Pre-condition** | Agent mode enabled |
| **Steps** | 1. Create Application with `syncPolicy.automated.prune: false, selfHeal: false`<br>2. Wait for initial sync<br>3. Manually edit the deployed `Deployment` on the managed cluster (e.g., change replica count)<br>4. Wait 5 minutes<br>5. Verify the manual change persists — ArgoCD does not revert it<br>6. Verify Application shows `OutOfSync` (live state differs from Git)<br>7. Verify hub accurately reflects the drift |
| **Expected** | Manual change persists; Application shows OutOfSync but does not auto-correct |
| **Result** | **NOT YET TESTED** |

```yaml
syncPolicy:
  automated:
    prune: false
    selfHeal: false
```

---

#### TC-SYNC-05: Sync with `CreateNamespace` Option

| Field | Value |
|-------|-------|
| **Objective** | Verify the agent respects `CreateNamespace=true` sync option to auto-create the target namespace if it does not exist, without requiring a companion policy to pre-provision it |
| **Pre-condition** | Agent mode enabled; target namespace does NOT exist on managed cluster; RBAC includes namespace creation permission |
| **Steps** | 1. Create Application with `syncPolicy.syncOptions: ["CreateNamespace=true"]`<br>2. Target a namespace that does not exist on the managed cluster<br>3. Verify namespace is auto-created during sync<br>4. Verify workload deploys in the new namespace<br>5. Verify hub shows synced status |
| **Expected** | Namespace auto-created; workload deployed; no manual namespace provisioning needed |
| **Result** | **NOT YET TESTED** |

```yaml
syncPolicy:
  automated:
    prune: false
    selfHeal: true
  syncOptions:
    - CreateNamespace=true
```

---

#### TC-SYNC-06: No-Delete GitOps — Create and Update Only (Combined Configuration)

Since the companion policy always runs as `cluster-admin`, **RBAC is not the layer to restrict deletion**. Instead, "no delete" must be configured at the ArgoCD Application and AppProject level. There are three layers that can enforce this, and they can be combined for defense-in-depth:

| Layer | Where to configure | What it controls |
|-------|-------------------|-----------------|
| **Application `syncPolicy`** | `syncPolicy.automated.prune: false` on the `Application` resource | Prevents ArgoCD from deleting resources when they are removed from the Git source |
| **Per-resource annotation** | `argocd.argoproj.io/sync-options: Prune=false` on individual Kubernetes resources in Git | Protects specific resources from pruning even if the Application has `prune: true` |
| **AppProject `orphanedResources`** | `spec.orphanedResources.warn: true` on the `AppProject` (without `ignore` patterns) | Warns about orphaned resources instead of allowing silent deletion; does not block prune but provides visibility |

> **Key point:** `prune: false` on the Application is the primary control. The per-resource annotation is a safety net for critical resources. Neither changes the SA's RBAC — the agent still has `cluster-admin` permissions but ArgoCD's application layer will not issue delete calls.

| Field | Value |
|-------|-------|
| **Objective** | Verify that a complete "no delete" configuration prevents any resource deletion via Git on the managed cluster through the agent, while still allowing create and update operations |
| **Pre-condition** | Agent mode enabled; hub principal configured |
| **Steps** | 1. Create AppProject `create-update-only` with `orphanedResources.warn: true`<br>2. Create Application referencing this project with `syncPolicy.automated.prune: false, selfHeal: true`<br>3. Git source contains: `Deployment`, `Service`, `ConfigMap` (annotated with `Prune=false`)<br>4. Wait for initial sync — verify all 3 resources created on managed cluster<br>5. **Test create:** Add a new `ConfigMap` (`config-extra`) to the Git source — verify it is created on managed cluster<br>6. **Test update:** Change the `Deployment` replica count in Git — verify it is updated on managed cluster<br>7. **Test no-delete (application-level):** Remove the `Service` from Git — verify it is NOT deleted from managed cluster; Application shows `OutOfSync`<br>8. **Test no-delete (annotation-level):** Temporarily set `prune: true` on the Application, then remove the annotated `ConfigMap` from Git — verify it is NOT deleted (annotation overrides)<br>9. **Test no-delete (manual sync with prune):** Run `argocd app sync --prune` manually — verify the `Service` (no annotation) IS pruned but the annotated `ConfigMap` is NOT<br>10. Verify hub accurately mirrors all states throughout |
| **Expected** | Creates and updates succeed; automated deletion is blocked by `prune: false`; per-resource annotation provides additional protection; `orphanedResources.warn` logs warnings for out-of-sync resources |
| **Result** | **NOT YET TESTED** |

```yaml
# 1. AppProject — enable orphaned resource warnings
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: create-update-only
  namespace: openshift-gitops
spec:
  sourceRepos:
    - "https://github.com/argoproj/argocd-example-apps"
  destinations:
    - namespace: "guestbook"
      server: "*"
  clusterResourceWhitelist: []
  namespaceResourceWhitelist:
    - group: ""
      kind: "*"
    - group: "apps"
      kind: "*"
  orphanedResources:
    warn: true
---
# 2. Application — prune: false, selfHeal: true (create + update only)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-no-delete
  namespace: openshift-gitops
spec:
  project: create-update-only
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps
    targetRevision: HEAD
    path: guestbook
  destination:
    name: '{{name}}'
    namespace: guestbook
  syncPolicy:
    automated:
      prune: false       # <-- primary control: never auto-delete
      selfHeal: true      # <-- still auto-correct drift (updates only)
    syncOptions:
      - CreateNamespace=true
---
# 3. Per-resource annotation (in the Git-managed manifests)
# Add to any resource that must NEVER be deleted, even during manual prune:
apiVersion: v1
kind: ConfigMap
metadata:
  name: critical-config
  namespace: guestbook
  annotations:
    argocd.argoproj.io/sync-options: Prune=false   # <-- per-resource safety net
data:
  key: value
```

**Configuration summary for "no delete via Git":**

```
Where to set it:
├── Application.spec.syncPolicy.automated.prune: false     ← REQUIRED (primary)
├── Resource annotation: Prune=false                        ← RECOMMENDED for critical resources
└── AppProject.spec.orphanedResources.warn: true            ← OPTIONAL (visibility)

What it does NOT affect:
├── RBAC (cluster-admin) — unchanged, agent can still delete if instructed
├── Manual kubectl delete — still works, this only controls ArgoCD behavior
└── ArgoCD UI/CLI manual prune — works unless resource has Prune=false annotation
```

---

#### TC-SYNC-07: No-Delete GitOps with ApplicationSet in Any Namespace

ApplicationSets introduce a **second deletion vector** beyond `prune`. When a cluster is removed from the Placement/generator, the ApplicationSet controller deletes the generated `Application`, which by default **cascade-deletes all its managed resources** on the spoke. `prune: false` in the template does NOT prevent this — it only controls what happens when a resource is removed from Git, not when the entire Application is removed by the ApplicationSet controller.

To achieve full "no delete" with ApplicationSets, you need both:

| Layer | Config | What it prevents |
|-------|--------|-----------------|
| **ApplicationSet `syncPolicy`** | `preserveResourcesOnDeletion: true` | Prevents cascade deletion of spoke resources when a generated Application is removed (e.g., cluster removed from Placement) |
| **Template `syncPolicy`** | `automated.prune: false` | Prevents deletion of individual resources when they are removed from the Git source |

> **Without `preserveResourcesOnDeletion: true`:** if a managed cluster is removed from the Placement, the ApplicationSet deletes the generated Application, which deletes all resources on that cluster — even though `prune: false` is set. This is because the Application itself is being deleted, not just a resource within it.

| Field | Value |
|-------|-------|
| **Objective** | Verify that an ApplicationSet deploying to any namespace across multiple managed clusters preserves all spoke resources when: (a) a resource is removed from Git, and (b) a cluster is removed from the Placement |
| **Pre-condition** | Agent mode enabled; hub principal with `allowedNamespaces: ["*"]`; Placement selecting 2+ managed clusters; custom AppProject allowing multiple destination namespaces |
| **Steps** | 1. Create AppProject `appset-no-delete` allowing destinations `namespace: "*"` with `orphanedResources.warn: true`<br>2. Apply ApplicationSet with `preserveResourcesOnDeletion: true` and template `syncPolicy.automated.prune: false, selfHeal: true`<br>3. Use `clusterDecisionResource` generator referencing `acm-placement`<br>4. Template deploys to namespace `app-{{name}}` (dynamic per cluster) with `CreateNamespace=true`<br>5. Wait for generated Applications to sync on all matched clusters<br>6. Verify workloads running in `app-<cluster>` namespace on each managed cluster<br>7. **Test no-delete (Git removal):** Remove a `ConfigMap` from the Git source — verify it is NOT deleted on any managed cluster<br>8. **Test no-delete (cluster removal):** Remove one cluster from the Placement label selector — verify the generated Application is deleted from hub, but **all resources remain on the spoke cluster**<br>9. **Test re-add:** Add the cluster back to the Placement — verify ApplicationSet regenerates the Application and it syncs without duplicating existing resources<br>10. Verify hub accurately mirrors all states throughout |
| **Expected** | Resources survive both Git-level removal (prune: false) and ApplicationSet-level Application removal (preserveResourcesOnDeletion: true); re-adding the cluster reconciles cleanly |
| **Result** | **NOT YET TESTED** |

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: appset-no-delete
  namespace: openshift-gitops
spec:
  sourceRepos:
    - "https://github.com/argoproj/argocd-example-apps"
  destinations:
    - namespace: "*"
      server: "*"
  clusterResourceWhitelist: []
  namespaceResourceWhitelist:
    - group: ""
      kind: "*"
    - group: "apps"
      kind: "*"
  orphanedResources:
    warn: true
---
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook-no-delete
  namespace: openshift-gitops
spec:
  generators:
    - clusterDecisionResource:
        configMapRef: acm-placement
        labelSelector:
          matchLabels:
            cluster.open-cluster-management.io/placement: agent-placement
        requeueAfterSeconds: 180
  syncPolicy:
    preserveResourcesOnDeletion: true    # <-- prevents cascade delete on Application removal
  template:
    metadata:
      name: 'guestbook-{{name}}'
    spec:
      project: appset-no-delete
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps
        targetRevision: HEAD
        path: guestbook
      destination:
        name: '{{name}}'
        namespace: 'app-{{name}}'        # <-- dynamic namespace per cluster
      syncPolicy:
        automated:
          prune: false                    # <-- never auto-delete resources from Git removal
          selfHeal: true
        syncOptions:
          - CreateNamespace=true          # <-- auto-create target namespace
```

**Full "no delete" configuration with ApplicationSets — reference:**

```
Where to set it:
├── ApplicationSet.spec.syncPolicy.preserveResourcesOnDeletion: true   ← REQUIRED
│   └── Prevents: cluster removed from Placement → Application deleted → resources deleted
├── Template.spec.syncPolicy.automated.prune: false                     ← REQUIRED
│   └── Prevents: resource removed from Git → resource deleted on spoke
├── Resource annotation: Prune=false                                    ← RECOMMENDED
│   └── Prevents: manual argocd app sync --prune from deleting critical resources
└── AppProject.spec.orphanedResources.warn: true                        ← OPTIONAL
    └── Provides: visibility into resources that exist on cluster but not in Git

Two deletion vectors in ApplicationSets:
1. Git-level removal  → controlled by prune: false (template syncPolicy)
2. Application removal → controlled by preserveResourcesOnDeletion: true (ApplicationSet syncPolicy)

Both must be set — they are independent controls.
```

---

### 2.3 — Principal `allowedNamespaces` Restriction Test

#### TC-PRINCIPAL-01: Hub Principal with Restricted `allowedNamespaces`

| Field | Value |
|-------|-------|
| **Objective** | Verify that the hub principal's `allowedNamespaces` configuration restricts which namespaces the agent can operate in |
| **Pre-condition** | Agent mode enabled |
| **Steps** | 1. Configure hub principal with `allowedNamespaces: ["openshift-gitops", "guestbook"]` (not `["*"]`)<br>2. Create Application targeting namespace `guestbook` — verify sync succeeds<br>3. Create Application targeting namespace `unauthorized-ns` — verify it is rejected by the principal<br>4. Verify hub reflects the denial |
| **Expected** | Allowed namespace works; disallowed namespace is rejected at the principal level |
| **Result** | **NOT YET TESTED** |

---

### 2.4 — Policy Configuration & Prerequisites Tests

#### TC-POLICY-01: Policy Customization Survives Controller Reconciliation

| Field | Value |
|-------|-------|
| **Objective** | Verify that user-added Policy templates (RBAC, namespaces, Applications) are preserved when the GitOpsCluster controller reconciles |
| **Pre-condition** | Agent mode enabled; Policy auto-generated by controller |
| **Steps** | 1. Wait for the controller to generate the ArgoCD Policy<br>2. Patch the Policy to add a custom ConfigurationPolicy with a ClusterRoleBinding, a Namespace, and a view ClusterRoleBinding for the agent SA<br>3. Trigger a controller reconcile (e.g., update a GitOpsCluster annotation)<br>4. Wait for reconcile to complete<br>5. Verify all user-added templates are still present in the Policy<br>6. Verify only `argoCDAgent.agent.image` was potentially touched (if drift heal ran)<br>7. Verify the Policy is still Compliant on managed clusters |
| **Expected** | All user customizations preserved; controller only touches agent image field |
| **Result** | **NOT YET TESTED** |

---

#### TC-POLICY-02: Policy Deletion and Recreation

| Field | Value |
|-------|-------|
| **Objective** | Verify the controller recreates a deleted Policy, and that `skip-argocd-policy` annotation prevents recreation |
| **Pre-condition** | Agent mode enabled; Policy exists and is Compliant |
| **Steps** | 1. Delete the ArgoCD Policy<br>2. Wait for controller reconcile<br>3. Verify the Policy is recreated with the ArgoCD CR template<br>4. Re-add user customizations (RBAC, namespaces) to the recreated Policy<br>5. Verify Policy becomes Compliant again<br>6. Set `apps.open-cluster-management.io/skip-argocd-policy: "true"` annotation on GitOpsCluster<br>7. Delete the Policy again<br>8. Wait for controller reconcile<br>9. Verify the Policy is NOT recreated<br>10. Verify `ArgoCDPolicyReady` condition shows `Reason=Skipped` |
| **Expected** | Without annotation: Policy recreated. With annotation: Policy stays deleted, condition shows Skipped |
| **Result** | **NOT YET TESTED** |

---

#### TC-POLICY-03: Missing Prerequisites — Clear Error Reporting

| Field | Value |
|-------|-------|
| **Objective** | Verify that missing each critical prerequisite produces a clear, actionable error rather than a silent failure |
| **Pre-condition** | Agent mode enabled |
| **Steps** | Test each prerequisite independently:<br>1. **Missing `ManagedClusterSetBinding`:** Delete the binding in `openshift-gitops` — verify Placement selects zero clusters and GitOpsCluster status reports the issue<br>2. **Missing `default` AppProject wildcards:** Remove wildcard `sourceNamespaces` from the `default` AppProject — verify the principal does not propagate the project; verify agents log "project not found" and Application fails<br>3. **Missing `ARGOCD_CLUSTER_CONFIG_NAMESPACES`:** Remove `local-cluster` from the env var — verify agent/principal pods crash with "namespaces is forbidden"; verify pod logs contain the error<br>4. **Missing `view` ClusterRoleBinding** for agent SA — verify ArgoCD UI live manifest returns "Resource not found in cluster" (not a generic error)<br>5. **`destinationBasedMapping` mismatch:** Set `destinationBasedMapping: true` on principal but omit on agent — verify ArgoCD UI shows "Resource not found in cluster" and agent logs show unexpected key format error |
| **Expected** | Each missing prerequisite produces a specific, identifiable error message — no silent failures |
| **Result** | **NOT YET TESTED** |

---

#### TC-POLICY-04: Policy with Agent SA `view` ClusterRoleBinding for Live Manifest

| Field | Value |
|-------|-------|
| **Objective** | Verify that adding a `view` ClusterRoleBinding for the ArgoCD agent SA to the Policy enables ArgoCD UI live manifest on managed clusters |
| **Pre-condition** | Agent mode enabled; Application synced and healthy |
| **Steps** | 1. Without the `view` binding — confirm ArgoCD UI live manifest fails ("Resource not found in cluster")<br>2. Add a `view` ClusterRoleBinding to the Policy as an object template:<br>`clusterrole: view`, `serviceaccount: openshift-gitops:acm-openshift-gitops-agent-agent`<br>3. Wait for Policy to become Compliant<br>4. Verify ArgoCD UI live manifest now works — resource tree and live state visible<br>5. Verify this works on both OCP and Kind managed clusters |
| **Expected** | Live manifest fails without binding; works after adding it via Policy; no manual kubectl needed |
| **Result** | **NOT YET TESTED** |

---

#### TC-POLICY-05: Policy with `destinationBasedMapping` Consistency

| Field | Value |
|-------|-------|
| **Objective** | Verify that setting `destinationBasedMapping: true` on the agent's ArgoCD CR (in the Policy) makes ArgoCD UI live manifest work when the principal also uses `destinationBasedMapping` |
| **Pre-condition** | Agent mode enabled; principal has `destinationBasedMapping: true` |
| **Steps** | 1. Verify principal ArgoCD CR has `destinationBasedMapping: true`<br>2. Modify the Policy to add `spec.argoCDAgent.agent.client.destinationBasedMapping: true` to the ArgoCD CR object template<br>3. Wait for Policy to become Compliant on managed clusters<br>4. Deploy an Application and wait for sync<br>5. Verify ArgoCD UI live manifest works — no "Resource not found" or Redis key format errors<br>6. Check agent logs for absence of "unexpected key format" errors |
| **Expected** | With matching `destinationBasedMapping`, UI live manifest works; Redis keys are consistent |
| **Result** | **NOT YET TESTED** |

---

#### TC-POLICY-06: Cert Rotation Resilience

| Field | Value |
|-------|-------|
| **Objective** | Verify the agent survives automatic cert rotation without manual intervention and without Application disruption |
| **Pre-condition** | Agent mode enabled; Applications synced and healthy; `argocd-agent-client-tls` cert has 24-hour validity |
| **Steps** | 1. Record current `argocd-agent-client-tls` cert data on managed cluster<br>2. Delete `argocd-agent-client-tls` secret on managed cluster to simulate rotation<br>3. Verify the `SecretReconciler` re-creates the secret within 120 seconds (from the source secret in `open-cluster-management-agent-addon`)<br>4. Verify agent pods are rolling-restarted (pod template annotation `apps.open-cluster-management.io/cert-rotated-at` updated)<br>5. Verify agent re-establishes mTLS connection to principal after restart<br>6. Verify existing Applications remain `Synced`/`Healthy` — no disruption<br>7. Verify hub status accurately reflects managed cluster state throughout |
| **Expected** | Secret re-created; agent restarted; mTLS restored; Applications unaffected |
| **Result** | **NOT YET TESTED** |

---

#### TC-POLICY-07: PlacementDecision `score` Workaround for ApplicationSet

| Field | Value |
|-------|-------|
| **Objective** | Verify the workaround for the OCM PlacementDecision `score:0` (int64) field that causes ArgoCD's ApplicationSet `clusterDecisionResource` DuckType generator to panic |
| **Pre-condition** | Agent mode enabled; Placement selecting 2+ clusters |
| **Steps** | 1. Check auto-generated PlacementDecision — verify it contains `score:0` (int64 field)<br>2. Attempt to create ApplicationSet pointing to the auto-generated PlacementDecision label — verify it panics or errors<br>3. Create a manual PlacementDecision with a **different label** containing only `clusterName` and `reason` (no `score`)<br>4. Create ApplicationSet pointing to the manual PlacementDecision's label<br>5. Verify Applications are generated correctly — no panic<br>6. Add/remove clusters from the manual PlacementDecision — verify ApplicationSet updates correctly |
| **Expected** | Auto-generated PlacementDecision causes panic; manual PlacementDecision (no `score`) works correctly |
| **Result** | **NOT YET TESTED** |

> **Note:** This is a known ArgoCD bug. The workaround (manual PlacementDecision with a different label) must be documented and tested because every production deployment using `clusterDecisionResource` will hit this.

---

## 3. AppProject Test Recommendations

The agent pull model introduces a unique enforcement boundary: AppProject definitions live on the hub (principal side), but reconciliation happens on the spoke (agent side). This makes AppProject enforcement through the agent a higher-risk area than in standard ArgoCD — a bug could mean the principal accepts an Application that the agent enforces differently, or vice versa. The following recommendations are ordered by risk to production users.

### P0 — Multi-Tenant Isolation (TC-PROJECT-04)

**Start here.** This is the most critical AppProject test for the agent model. When two teams share a managed cluster with different AppProjects (`team-a` scoped to `team-a-ns`, `team-b` scoped to `team-b-ns`), project isolation must hold through the entire hub-to-spoke flow. If `team-a` can accidentally or intentionally deploy into `team-b-ns`, that is a multi-tenancy breach. This scenario is also the most likely to surface agent-specific bugs because both the principal and the agent need to enforce the boundary consistently — a mismatch between the two could allow cross-project resource access that would never occur in a single-cluster ArgoCD setup.

### P0 — Restricted `sourceRepos` (TC-PROJECT-01)

The second priority. `sourceRepos` is the most fundamental AppProject restriction — it controls which Git repositories can supply manifests. In the agent model, the key question is: does the principal reject the Application before it reaches the agent, or does the agent enforce it locally? Either way the sync should be blocked, but knowing *where* the enforcement happens matters for security posture. If enforcement only happens agent-side, a compromised or misconfigured agent could bypass it entirely, allowing manifests from unauthorized repositories.

### P1 — Deny Secrets via `namespaceResourceBlacklist` (TC-PROJECT-02)

This is the most common real-world AppProject restriction. Teams want Deployments, Services, and ConfigMaps managed via Git, but Secrets managed via Vault, External Secrets Operator, or sealed-secrets — never stored in Git. If the agent does not respect `namespaceResourceBlacklist`, secrets leak into Git workflows. This also validates that the agent correctly handles partial sync failures (Deployment succeeds, Secret is denied).

### P2 — Empty `clusterResourceWhitelist` (TC-PROJECT-03)

Lower priority. Most setups with cluster-admin RBAC rely on AppProject as the guardrail against cluster-scoped resource creation (Namespaces, ClusterRoles, ClusterRoleBindings). Since the companion policy correctly runs as cluster-admin, the AppProject is the only layer preventing Git-driven creation of cluster-scoped resources. Worth testing but less urgent than the multi-tenancy and sourceRepos scenarios, because accidental cluster-scoped resource creation is typically caught during code review of the Git repo.

---

## 4. Proposed Test Priority (Full Matrix)

| Priority | Test ID | Rationale |
|----------|---------|-----------|
| **P0 — Must have** | TC-PROJECT-04 | Multi-tenant isolation through the agent — highest risk for cross-project breach |
| **P0 — Must have** | TC-PROJECT-01 | sourceRepos restriction — fundamental AppProject guardrail, enforcement boundary unclear in agent model |
| **P0 — Must have** | TC-POLICY-03 | Missing prerequisites — verify clear errors for ManagedClusterSetBinding, AppProject wildcards, ARGOCD_CLUSTER_CONFIG_NAMESPACES, view binding, DBM mismatch |
| **P0 — Must have** | TC-POLICY-07 | PlacementDecision `score` workaround — every production ApplicationSet with `clusterDecisionResource` will hit this |
| **P0 — Must have** | TC-APPSET-03 | MatrixGenerator (ClusterDecision + Git) — the primary multi-cluster + multi-app pattern for agent pull model |
| **P0 — Must have** | TC-APPSET-04 | Cluster disconnected — agent resilience, workload survival, reconnection re-sync |
| **P0 — Must have** | TC-SYNC-07 | No-delete with ApplicationSet in any namespace — covers both deletion vectors (Git removal + Application removal via Placement) |
| **P0 — Must have** | TC-SYNC-06 | Full "no delete via Git" test — combines `prune: false`, per-resource annotation, and AppProject `orphanedResources` |
| **P0 — Must have** | TC-SYNC-01 | `prune: false` is the most common production pattern for safe GitOps |
| **P1 — Should have** | TC-POLICY-01 | Policy customization survives controller reconcile — validates user-owned Policy contract |
| **P1 — Should have** | TC-POLICY-02 | Policy deletion/recreation + skip-argocd-policy annotation |
| **P1 — Should have** | TC-POLICY-04 | Agent SA `view` ClusterRoleBinding via Policy — enables ArgoCD UI live manifest |
| **P1 — Should have** | TC-POLICY-05 | destinationBasedMapping consistency via Policy — prevents Redis key mismatch |
| **P1 — Should have** | TC-POLICY-06 | Cert rotation resilience — 24h cert lifecycle, agent restart, no Application disruption |
| **P1 — Should have** | TC-PROJECT-02 | Blocking Secrets via Git is a common security policy; validates partial sync failure handling |
| **P1 — Should have** | TC-SYNC-04 | selfHeal: false is critical for teams that make manual cluster-side changes |
| **P1 — Should have** | TC-SYNC-03 | Per-resource prune protection is a widely-used safety mechanism |
| **P1 — Should have** | TC-APPSET-01 | GitGenerator (directories) — common pattern for mono-repo GitOps |
| **P1 — Should have** | TC-APPSET-06 | Cluster disconnected + removed from Placement — worst-case resilience scenario |
| **P1 — Should have** | TC-APPSET-05 | Cluster OCP upgrade — agent continuity and version drift heal |
| **P2 — Nice to have** | TC-PROJECT-03 | clusterResourceWhitelist enforcement — AppProject as guardrail for cluster-scoped resources |
| **P2 — Nice to have** | TC-APPSET-02 | GitGenerator (files) — parameterized per-cluster config from Git |
| **P2 — Nice to have** | TC-SYNC-02 | Manual-only sync |
| **P2 — Nice to have** | TC-SYNC-05 | CreateNamespace sync option |
| **P2 — Nice to have** | TC-PRINCIPAL-01 | Principal namespace restriction |

---

## 5. Summary

The current test plan validates the **happy path with wide-open AppProject, aggressive sync policies, and default Policy configuration**. It proves the agent pull model works when:
- RBAC is `cluster-admin` (correct — companion policy always runs as cluster-admin)
- AppProject is `default` (allow everything)
- Sync policy is `automated + prune + selfHeal`
- Policy uses the bare minimum auto-generated template
- All prerequisites are correctly configured

It does **not** validate:
- **Policy lifecycle:** customization survival, deletion/recreation, `skip-argocd-policy` annotation
- **Missing prerequisites:** silent failures when ManagedClusterSetBinding, AppProject wildcards, `ARGOCD_CLUSTER_CONFIG_NAMESPACES`, agent SA `view` binding, or `destinationBasedMapping` are misconfigured
- **PlacementDecision `score` bug:** every `clusterDecisionResource` ApplicationSet hits this in production
- **Cert rotation:** 24-hour cert lifecycle and agent restart behavior
- Custom AppProjects restricting repos, namespaces, or resource kinds
- `prune: false` (create/update only, never delete)
- `selfHeal: false` (allow manual cluster changes)
- ApplicationSet generators beyond `clusterDecisionResource` (GitGenerator, MatrixGenerator)
- Cluster disconnection/reconnection and OCP upgrade resilience

**Recommendation:** Prioritize TC-POLICY-03 (missing prerequisites), TC-POLICY-07 (PlacementDecision workaround), TC-PROJECT-04 (multi-tenant isolation), and TC-SYNC-01 (prune: false) as immediate additions. TC-POLICY-03 and TC-POLICY-07 catch issues that every production deployment will encounter on first setup. TC-PROJECT-04 and TC-PROJECT-01 target the agent-specific enforcement boundary. TC-SYNC-01 covers the most common production sync policy.
