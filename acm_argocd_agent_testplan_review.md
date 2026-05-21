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

### 1.3 — Other Gaps

| Gap | Impact |
|-----|--------|
| **No `allowedNamespaces` restriction test** — hub principal is configured with `allowedNamespaces: ["*"]` in every test | Unknown whether namespace restrictions on the principal are enforced through the agent |
| **No agent version drift heal test** — the doc describes auto-patching agent images when the principal upgrades, but no test validates it | Upgrade scenarios untested |
| **No network disruption / reconnection test** — what happens when the agent loses mTLS connection to the principal and reconnects? | Resilience unknown |
| **No multi-AppProject test** — what if two Applications in different AppProjects target the same managed cluster? | Project isolation on the agent side untested |
| **No `syncPolicy.syncOptions` coverage** — `CreateNamespace`, `ServerSideApply`, `FailOnSharedResource` | Agent may behave differently than direct controller for these options |

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

### 2.2 — Sync Policy / Prune Protection Tests

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
| **P0 — Must have** | TC-SYNC-07 | No-delete with ApplicationSet in any namespace — covers both deletion vectors (Git removal + Application removal via Placement) |
| **P0 — Must have** | TC-SYNC-06 | Full "no delete via Git" test — combines `prune: false`, per-resource annotation, and AppProject `orphanedResources` |
| **P0 — Must have** | TC-SYNC-01 | `prune: false` is the most common production pattern for safe GitOps |
| **P1 — Should have** | TC-PROJECT-02 | Blocking Secrets via Git is a common security policy; validates partial sync failure handling |
| **P1 — Should have** | TC-SYNC-04 | selfHeal: false is critical for teams that make manual cluster-side changes |
| **P1 — Should have** | TC-SYNC-03 | Per-resource prune protection is a widely-used safety mechanism |
| **P2 — Nice to have** | TC-PROJECT-03 | clusterResourceWhitelist enforcement — AppProject as guardrail for cluster-scoped resources |
| **P2 — Nice to have** | TC-SYNC-02 | Manual-only sync |
| **P2 — Nice to have** | TC-SYNC-05 | CreateNamespace sync option |
| **P2 — Nice to have** | TC-PRINCIPAL-01 | Principal namespace restriction |

---

## 4. Summary

The current test plan validates the **happy path with wide-open AppProject and aggressive sync policies**. It proves the agent pull model works when:
- RBAC is `cluster-admin` (correct — companion policy always runs as cluster-admin)
- AppProject is `default` (allow everything)
- Sync policy is `automated + prune + selfHeal`

It does **not** validate configurations that production users commonly need:
- Custom AppProjects restricting repos, namespaces, or resource kinds
- `prune: false` (create/update only, never delete)
- `selfHeal: false` (allow manual cluster changes)
- Per-resource sync option overrides

**Recommendation:** Prioritize TC-PROJECT-04 (multi-tenant isolation), TC-PROJECT-01 (sourceRepos), and TC-SYNC-01 (prune: false) as immediate additions. TC-PROJECT-04 and TC-PROJECT-01 target the agent-specific enforcement boundary where hub principal and spoke agent must agree — the highest-risk area unique to the pull model. TC-SYNC-01 covers the most common production sync policy divergence.
