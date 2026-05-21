# Test Plan Critical Review & Additional Test Cases

**Reviewing:** `acm_argocd_agent_testplan.md`
**Date:** 2026-05-21

---

## 1. Critical Findings

### 1.1 — RBAC: Every Test Uses `cluster-admin`

The companion policy in the current test plan (and the PDF report) binds the ArgoCD application controller ServiceAccount to `cluster-admin`:

```yaml
roleRef:
  kind: ClusterRole
  name: cluster-admin    # <-- this is what every test relies on
subjects:
  - kind: ServiceAccount
    name: acm-openshift-gitops-argocd-application-controller
    namespace: openshift-gitops
```

**What's missing:**
- No test with a **namespace-scoped Role** instead of a ClusterRole — what happens when the agent SA can only deploy into specific namespaces?
- No test with a **least-privilege ClusterRole** (e.g., only `create`, `get`, `list`, `update`, `patch` on specific resource kinds, no `delete`)
- No **negative test** verifying that the agent correctly fails when RBAC denies an operation (e.g., trying to create a `ClusterRole` when only namespace resources are permitted)
- No test for RBAC differences between the agent SA and the application controller SA — these are distinct principals in the agent model

**Risk:** Every customer deployment guide says "don't use cluster-admin in production." If we only test with cluster-admin, we have zero confidence that least-privilege setups work.

---

### 1.2 — Default `AppProject` Used Almost Everywhere

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

### 1.3 — All Sync Policies Use `prune: true` — No Create/Update-Only Scenario

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

### 1.4 — Other Gaps

| Gap | Impact |
|-----|--------|
| **No `allowedNamespaces` restriction test** — hub principal is configured with `allowedNamespaces: ["*"]` in every test | Unknown whether namespace restrictions on the principal are enforced through the agent |
| **No agent version drift heal test** — the doc describes auto-patching agent images when the principal upgrades, but no test validates it | Upgrade scenarios untested |
| **No network disruption / reconnection test** — what happens when the agent loses mTLS connection to the principal and reconnects? | Resilience unknown |
| **No multi-AppProject test** — what if two Applications in different AppProjects target the same managed cluster? | Project isolation on the agent side untested |
| **No `syncPolicy.syncOptions` coverage** — `CreateNamespace`, `ServerSideApply`, `FailOnSharedResource` | Agent may behave differently than direct controller for these options |

---

## 2. Additional Test Cases

### 2.1 — RBAC Tests

#### TC-RBAC-01: Least-Privilege Namespace-Scoped RBAC

| Field | Value |
|-------|-------|
| **Objective** | Verify agent pull model works when the application controller SA has only a namespace-scoped `Role` (not `ClusterRole`) limited to a single target namespace |
| **Pre-condition** | Agent mode enabled; companion policy deploys a `Role` + `RoleBinding` in `guestbook` namespace only (not a `ClusterRoleBinding`) |
| **Steps** | 1. Create companion policy that provisions a `Role` in `guestbook` namespace with verbs: `get`, `list`, `create`, `update`, `patch` on `deployments`, `services`, `pods`<br>2. Bind the role to the argocd application controller SA<br>3. Create `Application` on hub targeting `guestbook` namespace<br>4. Verify application syncs and workload deploys<br>5. Verify the agent does NOT have permissions outside `guestbook` namespace |
| **Expected** | Application syncs successfully within the permitted namespace; operations outside it are denied |
| **Result** | **NOT YET TESTED** |

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: gitops-guestbook-deployer
  namespace: guestbook
rules:
  - apiGroups: ["", "apps"]
    resources: ["deployments", "services", "pods", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: gitops-guestbook-deployer-binding
  namespace: guestbook
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: gitops-guestbook-deployer
subjects:
  - kind: ServiceAccount
    name: acm-openshift-gitops-argocd-application-controller
    namespace: openshift-gitops
```

---

#### TC-RBAC-02: RBAC Denies Cluster-Scoped Resource Creation

| Field | Value |
|-------|-------|
| **Objective** | Verify that an Application attempting to create cluster-scoped resources (e.g., `ClusterRole`, `Namespace`) fails gracefully when RBAC only grants namespace-scoped permissions |
| **Pre-condition** | Agent mode enabled; application controller SA has only namespace-scoped `Role` (no `ClusterRole`) |
| **Steps** | 1. Use the namespace-scoped Role from TC-RBAC-01<br>2. Create an Application whose Git source includes a `ClusterRole` and a `Namespace` resource<br>3. Observe sync behavior<br>4. Verify Application enters `Degraded` or `SyncFailed` status on the managed cluster<br>5. Verify hub reflects the failure status accurately |
| **Expected** | Sync fails for cluster-scoped resources with RBAC denial error; namespace-scoped resources in the same app may partially sync or the entire sync fails (document actual behavior); hub mirrors the error state |
| **Result** | **NOT YET TESTED** |

---

#### TC-RBAC-03: RBAC Without Delete Permissions

| Field | Value |
|-------|-------|
| **Objective** | Verify that even with `prune: true`, the agent cannot delete resources when the SA lacks `delete` verb in RBAC |
| **Pre-condition** | Agent mode enabled; companion policy grants `get`, `list`, `create`, `update`, `patch` but NOT `delete` |
| **Steps** | 1. Deploy Application with `prune: true` and `selfHeal: true`<br>2. Wait for initial sync to complete<br>3. Remove a resource from the Git source<br>4. Wait for ArgoCD to detect the drift and attempt pruning<br>5. Verify prune fails with RBAC denial<br>6. Verify the resource remains on the managed cluster<br>7. Verify hub shows the prune failure status |
| **Expected** | Resource is NOT deleted; Application shows `OutOfSync` with prune error; RBAC denial logged |
| **Result** | **NOT YET TESTED** |

---

#### TC-RBAC-04: Least-Privilege ClusterRole (No Wildcard)

| Field | Value |
|-------|-------|
| **Objective** | Verify agent works with a `ClusterRole` that explicitly lists allowed API groups, resources, and verbs instead of using wildcards |
| **Pre-condition** | Agent mode enabled |
| **Steps** | 1. Create a `ClusterRole` that permits only: `apps/deployments`, `core/services`, `core/configmaps`, `core/namespaces` with verbs `get`, `list`, `watch`, `create`, `update`, `patch`<br>2. Bind to application controller SA via `ClusterRoleBinding`<br>3. Deploy Application that uses only these resource types<br>4. Verify sync succeeds<br>5. Deploy a second Application that attempts to create a `Secret` (not in the allowed list)<br>6. Verify the second application fails |
| **Expected** | First application syncs; second application fails with Forbidden error |
| **Result** | **NOT YET TESTED** |

---

### 2.2 — AppProject Enforcement Tests

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

### 2.4 — Principal `allowedNamespaces` Restriction Test

#### TC-PRINCIPAL-01: Hub Principal with Restricted `allowedNamespaces`

| Field | Value |
|-------|-------|
| **Objective** | Verify that the hub principal's `allowedNamespaces` configuration restricts which namespaces the agent can operate in |
| **Pre-condition** | Agent mode enabled |
| **Steps** | 1. Configure hub principal with `allowedNamespaces: ["openshift-gitops", "guestbook"]` (not `["*"]`)<br>2. Create Application targeting namespace `guestbook` — verify sync succeeds<br>3. Create Application targeting namespace `unauthorized-ns` — verify it is rejected by the principal<br>4. Verify hub reflects the denial |
| **Expected** | Allowed namespace works; disallowed namespace is rejected at the principal level |
| **Result** | **NOT YET TESTED** |

---

## 3. Proposed Test Priority

| Priority | Test ID | Rationale |
|----------|---------|-----------|
| **P0 — Must have** | TC-SYNC-01 | `prune: false` is the most common production pattern for safe GitOps |
| **P0 — Must have** | TC-RBAC-01 | Least-privilege RBAC is a security requirement for every production deployment |
| **P0 — Must have** | TC-PROJECT-01 | sourceRepos restriction is a baseline AppProject use case |
| **P1 — Should have** | TC-RBAC-03 | RBAC without delete verbs is the server-side enforcement of "no deletion via Git" |
| **P1 — Should have** | TC-SYNC-04 | selfHeal: false is critical for teams that make manual cluster-side changes |
| **P1 — Should have** | TC-PROJECT-02 | Blocking Secrets via Git is a common security policy |
| **P1 — Should have** | TC-SYNC-03 | Per-resource prune protection is a widely-used safety mechanism |
| **P2 — Nice to have** | TC-RBAC-02 | Negative test for cluster-scoped resource denial |
| **P2 — Nice to have** | TC-RBAC-04 | Non-wildcard ClusterRole validation |
| **P2 — Nice to have** | TC-PROJECT-03 | clusterResourceWhitelist enforcement |
| **P2 — Nice to have** | TC-PROJECT-04 | Multi-project isolation |
| **P2 — Nice to have** | TC-SYNC-02 | Manual-only sync |
| **P2 — Nice to have** | TC-SYNC-05 | CreateNamespace sync option |
| **P2 — Nice to have** | TC-PRINCIPAL-01 | Principal namespace restriction |

---

## 4. Summary

The current test plan validates the **happy path with maximum permissions**. It proves the agent pull model works when:
- RBAC is `cluster-admin`
- AppProject is `default` (allow everything)
- Sync policy is `automated + prune + selfHeal`

It does **not** validate any of the configurations real production users will run:
- Least-privilege RBAC (namespace-scoped or resource-restricted)
- Custom AppProjects restricting repos, namespaces, or resource kinds
- `prune: false` (create/update only, never delete)
- `selfHeal: false` (allow manual cluster changes)
- Per-resource sync option overrides

**Recommendation:** Prioritize TC-SYNC-01, TC-RBAC-01, and TC-PROJECT-01 as immediate additions — they represent the most common production divergence from the current test matrix.
