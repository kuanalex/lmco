# LMCO Internal Cluster Upgrade Summary: 5.1.2 to 5.3.1

## Executive Summary

Successfully completed upgrade from version 5.1.2 to 5.3.1 on LMCO Internal Cluster. The upgrade encountered **5 issues** requiring manual intervention.

- **Overall Status:** Completed with manual workarounds
- **Total Components Upgraded:** cpd_platform, scheduler, wkc, mantaflow, analyticsengine, datastage_ent, datastage_ent_plus, dv, dp, ws, ws_runtimes, wml, ws_pipelines, dmc, db2wh, spss, cognos_analytics, planning_analytics
- **Total Effort:** Approximately 13 hours over 2 business days (March 13 & 16, 2026)
  - **Day 1 (March 13):** Approximately 8.5 hours (9:30 AM - 6:00 PM EST)
  - **Day 2 (March 16):** Approximately 4.5 hours (8:30 AM - 1:00 PM EST)
- **Known Outstanding Issues:** 2 issues remain unresolved (Tekton/Pipelines deployment and DV service instance)

## Upgrade Timeline & Duration

#### Day 1 - March 13, 2026 (Approximately 8.5 hours: 9:30 AM - 6:00 PM EST)

| Phase | Start Time | Duration | Status | Notes |
|-------|------------|----------|--------|-------|
| **Pre-upgrade Setup** | 9:30 AM | Approximately 1h 20m | Complete | Client workstation, environment variables, pre-checks, licensing, scheduler, entitlements, cluster scoped resources |
| **Shared Cluster Components** | 10:50 AM | 3m 6s | Complete | Clean upgrade |
| **Scheduler Upgrade** | - | 2m 49s | Complete | Clean upgrade |
| **Apply Entitlement** | - | 5m 30s | Complete | License entitlement applied |
| **Platform Operators Install** | 10:50 AM | 86m 40s | Failed | ZenService blocked by Tekton/Pipelines issue; monitoring until 12:14 PM |
| **Issue Investigation & Meetings** | 12:14 PM | 1h 28m | - | First meeting (26m 53s) + investigation + second meeting (1m 49s) |
| **ZenService Manual Patch** | 12:42 PM | 44m | Complete | Manually patched version to 6.4.0; reconciliation completed by 1:26 PM |
| **Image Digest Removal (Pre-emptive)** | Afternoon | Approximately 10 min | Complete | Removed digests from CCS, AE, WKC, Policy CRs |
| **Db2aaService Operator Upgrade** | Afternoon | 2m 8s | Complete | Manual workaround required (same as 5.3.0) |
| **WKC/DP/Mantaflow Attempt 1** | Afternoon | 28m 50s | Failed | WKC sub-CRs blocked by image_digests issue |
| **WKC Sub-CR Image Digest Removal** | Late Afternoon | Approximately 2 hours | Manual | Removed digests from 8 WKC sub-CRs + pinned images |
| **WKC/DP/Mantaflow Completion** | Over weekend | - | Complete | WKC completed after all sub-CR fixes |

### Day 2 - March 16, 2026 (Approximately 4.5 hours)

| Phase | Duration | Status | Notes |
|-------|----------|--------|-------|
| **DP/Mantaflow Standalone** | 15m 45s | Complete | Clean upgrade after WKC completion |
| **DMC/DB2WH/DV** | 21m 49s | Failed | DV failed due to image_digests |
| **DV Image Digest Removal** | Approximately 20 min | Complete | Patched DV CR and restarted operator |
| **WS/WS_Runtimes/WML/Pipelines** | 38m 40s | Failed | WS and WML had image_digests |
| **WS/WML Image Digest Removal** | Approximately 10 min | Complete | Patched both CRs |
| **WML Completion** | Approximately 40 min | Complete | WML reconciled after patch |
| **RStudio/SPSS/CA/PA** | 20m | Complete | Clean upgrade (RStudio not installed) |
| **Service Instance Fixes** | Approximately 1 hour | Partial | DMC instance fixed; DV instance remains in progress |
| **Post-Upgrade Migrations** | Approximately 30 min | Complete | CCS and IKC migrations successful |

## Issues Encountered

### Issue #1: ZenService CR Does Not Auto-Upgrade
**Severity:** High | **Component:** ZenService

During platform upgrade, the ZenService CR failed to automatically upgrade to version 6.4.0 and remained at the previous version, blocking the platform upgrade completion.

**Workaround:**
```bash
# Manually patch ZenService version
oc patch zenservice lite-cr -n zen --type=merge \
  --patch '{"spec":{"version":"6.4.0","zen_pak_version":"6.4.0"}}'
```
**Impact:** 86m platform install failure + 44m manual reconciliation. **Recommendation:** Manually patch ZenService CR version during platform upgrade.

### Issue #2: WKC and Sub-CRs Image Digests Not Auto-Removed
**Severity:** High | **Component:** WKC and all sub-CRs

The WKC upgrade failed with error: `No variable found with this name: wdp_profiling_messaging_image`. This affected not just the main WKC CR but also 8 sub-CRs (Policy, Glossary, Workflow, Profiling, DataQuality, Enrichment, Finley, WKCGovUI, KnowledgeGraph). The issue was caused by the operator's `/opt/ansible/5.3.1/roles/wkc-core/roles/common/tasks/hot_fix.yaml` attempting to reference `wdp_profiling_messaging_image` from the `image_digests` section, but this variable no longer existed in the 5.3.1 spec.

**Error Message:**
```
The task includes an option with an undefined variable. No variable found with this name: wdp_profiling_messaging_image

The error appears to be in '/opt/ansible/5.3.1/roles/wkc-core/roles/common/tasks/hot_fix.yaml': line 25, column 5
```

Confirmed with Carlos Renteria that wdp profiling messaging image is not valid and needed to be removed.

**Workaround:**
```bash
# Remove image_digests from all WKC-related CRs
oc patch wkc wkc-cr -n zen --type=json --patch '[{"op":"remove","path":"/spec/image_digests"}]'
oc patch policy policy-cr -n zen --type=json --patch '[{"op":"remove","path":"/spec/image_digests"}]'
oc patch glossary glossary-cr -n zen --type=json --patch '[{"op":"remove","path":"/spec/image_digests"}]'
oc patch workflow workflow-cr -n zen --type=json --patch '[{"op":"remove","path":"/spec/image_digests"}]'
oc patch profiling profiling-cr -n zen --type=json --patch '[{"op":"remove","path":"/spec/image_digests"}]'
oc patch dataquality data-quality-cr -n zen --type=json --patch '[{"op":"remove","path":"/spec/image_digests"}]'
oc patch enrichment enrichment-cr -n zen --type=json --patch '[{"op":"remove","path":"/spec/image_digests"}]'
oc patch finley finley-cr -n zen --type=json --patch '[{"op":"remove","path":"/spec/image_digests"}]'
oc patch wkcgovui wkc-gov-ui-cr -n zen --type=json --patch '[{"op":"remove","path":"/spec/image_digests"}]'
oc patch knowledgegraph knowledgegraph-cr -n zen --type=json --patch '[{"op":"remove","path":"/spec/image_digests"}]'

# Remove pinned images from specific CRs
oc patch wkc wkc-cr -n zen --type=json -p='[
  {"op":"remove","path":"/spec/wdp_profiling_ui_image"},
  {"op":"remove","path":"/spec/wkc_data_quality_ui_image"},
  {"op":"remove","path":"/spec/wkc_data_rules_image"},
  {"op":"remove","path":"/spec/wkc_glossary_service_image"},
  {"op":"remove","path":"/spec/wkc_gov_ui_image"}
]'
oc patch glossary glossary-cr -n zen --type=json -p='[{"op":"remove","path":"/spec/wkc_glossary_service_image"}]'
oc patch profiling profiling-cr -n zen --type=json -p='[{"op":"remove","path":"/spec/wdp_profiling_ui_image"}]'
oc patch dataquality data-quality-cr -n zen --type=json --patch '[
  {"op":"remove","path":"/spec/wkc_data_quality_ui_image"},
  {"op":"remove","path":"/spec/wkc_data_rules_image"}
]'
```
**Impact:** Approximately 2 hours of manual intervention across multiple CRs. **Recommendation:** Remove all image_digests and pinned images from WKC and sub-CRs before upgrade.

### Issue #3: DV Service Image Digests Not Auto-Removed
**Severity:** High | **Component:** Data Virtualization

DV upgrade failed due to pinned 5.1.2 image digests not being removed automatically.

**Workaround:**
```bash
oc patch dvservice dv-service -n zen --type=json -p='[{"op":"remove","path":"/spec/image_digests"}]'
oc delete pod -n cpd-operators -l app.kubernetes.io/name=ibm-dv-operator
```
**Impact:** 21m failed upgrade + 20m manual fix. **Recommendation:** Remove DV image_digests before upgrade.

### Issue #4: WS and WML Image Digests Not Auto-Removed
**Severity:** High | **Component:** Watson Studio, Watson Machine Learning

Both WS and WML CRs had pinned image digests that blocked upgrade completion.

**Workaround:**
```bash
oc patch ws ws-cr -n zen --type=json -p='[{"op":"remove","path":"/spec/image_digests"}]'
oc patch wmlbase wml-cr -n zen --type=json -p='[{"op":"remove","path":"/spec/image_digests"}]'
```
**Impact:** 38m failed upgrade + 10m manual fix + 40m WML reconciliation. **Recommendation:** Remove image_digests from WS and WML before upgrade.

### Issue #5: DMC Service Instance Does Not Auto-Upgrade
**Severity:** High | **Component:** DMC

DMC service instance remained stuck after CR upgrade due to invalid scaleConfig and zen-watcher sync issues.

**Workaround:**
```bash
# Fix invalid scaleConfig
oc patch dmc data-management-console -n zen --type=merge \
  -p '{"spec":{"scaleConfig":"small"}}'
# Force re-reconciliation
oc rollout restart deployment ibm-dmc-controller-manager -n cpd-operators
# Force Zen metastore re-sync
oc delete po -n zen -l app=zen-watcher
```
**Impact:** Approximately 1 hour manual intervention. **Recommendation:** Confirm DMC CR has valid scaleConfig and restart controller + zen-watcher after CR upgrade completes.

### Outstanding Issue #1: Tekton Extensions Controller CrashLoopBackOff
**GitHub Issue:** [#80026](https://github.ibm.com/PrivateCloud-analytics/CPD-Quality/issues/80026) | **Severity:** High | **Component:** WSPipelines, Tekton

The `tekton-extensions-controller` pod enters CrashLoopBackOff immediately after startup during upgrade from 5.1.2 to 5.3.1. The controller attempts to disable the `APIPriorityAndFairness` feature gate, but this feature is locked to true in the Kubernetes cluster.

**Error Message:**
```
{"level":"fatal","ts":1773679551.7298596,"caller":"controller/main.go:338",
"message":"Error setting features: cannot set feature gate APIPriorityAndFairness to false,
feature is locked to true"}
```

**Root Cause:** The tekton-extensions-controller attempts to modify a Kubernetes feature gate that is locked by the cluster, causing the pod to crash repeatedly. This prevents the WSPipelines CR from completing its upgrade, leaving it stuck at version 5.1.2 with 55% progress.

**Status:** Development team engaged for resolution.

**Impact:** WSPipelines CR cannot complete upgrade. **Recommendation:** Monitor for next actions from Pipelines development team.

### Outstanding Issue #2: DV Service Instance Does Not Auto-Upgrade
**Severity:** High | **Component:** Data Virtualization

After completing the DV CR upgrade, the DV service instance shows upgrade option to 3.3.1 but remains in PROVISIONED state at 3.1.2.

**Status:** Development team engaged for resolution.

**Impact:** Service instance not fully upgraded despite CR completion. **Recommendation:** Monitor for next actions from DV development team.