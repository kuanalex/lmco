# LMCO Internal Cluster Upgrade Summary: 5.1.2 → 5.3.0

## Executive Summary

Successfully completed upgrade from version 5.1.2 to 5.3.0 on LMCO Internal Cluster. The upgrade encountered **6 issues** requiring manual intervention.

- **Overall Status:** Completed with manual workarounds
- **Total Components Upgraded:** 20+ components including WKC, DataStage, Watson Studio, WML, DMC, and more
- **Total Effort:** ~11 hours over 3 business days (Feb 23-25, 2026)
- **Command Execution Time:** ~6.5 hours (CLI command runtime)
- **Troubleshooting & Manual Fixes:** ~4.5 hours (issue investigation, workarounds)

---

## Upgrade Timeline & Duration

### Day 1 - February 23, 2026 (~2 hours)

| Phase | Duration | Status | Notes |
|-------|----------|--------|-------|
| **Pre-upgrade Setup** | ~30 min | ✅ Complete | Client workstation, environment variables, pre-checks |
| **Shared Cluster Components** | 3m 26s | ✅ Complete | Timing issue on first attempt, retry successful |
| **Scheduler Upgrade** | 5m 8s | ✅ Complete | Clean upgrade |
| **CASE Package Download** | 5m | ✅ Complete | All component case bundles |
| **Platform Operators Install** | 85m 8s | ✅ Complete | Initial platform and operators |

### Day 2 - February 24, 2026 (~6 hours)

| Phase | Duration | Status | Issues |
|-------|----------|--------|--------|
| **WKC/DP/Mantaflow Attempt 1** | 89m 40s | ❌ Failed | AE, Db2aaService, Policy issues |
| **Issue Investigation & Fixes** | ~1 hours | ⚠️ Manual | Image digest removal, operator upgrade |
| **Db2aaService Operator Upgrade** | 11m 52s | ✅ Complete | Manual workaround required |
| **WKC/DP/Mantaflow Retry** | 121m 40s | ✅ Complete | After all fixes applied |
| **DP Standalone** | 18m 35s | ✅ Complete | Clean upgrade |
| **DMC/DB2WH/DV** | 30m 22s | ✅ Complete | Clean upgrade |
| **DataStage/WS/WML/Pipelines** | 20m | ✅ Complete | Canvas operator restart needed |

### Day 3 - February 25, 2026 (~3 hours)

| Phase | Duration | Status | Notes |
|-------|----------|--------|-------|
| **Service Instance Upgrades** | ~70 min | ✅ Complete | Spark (31s), DV (70 min) |
| **DMC Instance Investigation** | ~1 hours | ⚠️ Ongoing | Instance stuck in UPGRADE_IN_PROGRESS |
| **Documentation & Reporting** | ~1 hour | ✅ Complete | Issue logging, summary creation |

---

## Critical Issues Encountered

### Issue #1: AnalyticsEngine Image Digests Not Auto-Removed
**GitHub Issue:** [#77755](https://github.ibm.com/PrivateCloud-analytics/CPD-Quality/issues/77755)

**Severity:** 🔴 Critical - Blocked WKC upgrade at 85%

**Component:** AnalyticsEngine (AE)  
**Affected Pod:** `spark-hb-nginx`  
**Status:** CrashLoopBackOff (16 restarts over 64 minutes)

#### Problem Description
AnalyticsEngine CR upgrade failed with pinned 5.1.2 image digests remaining, causing spark-hb-nginx pod to crash (16 restarts over 64 minutes).

#### Root Cause
The `auto_hotfix_removal` feature is not supported by the AnalyticsEngine operator. Despite documentation suggesting this feature exists, the Spark team confirmed it was never implemented. Pinned 5.1.2 image digests remained after upgrade, causing incompatibility with 5.3.0 infrastructure.

#### Workaround Applied
```bash
oc patch ae analyticsengine-sample -n zen --type=json \
  --patch '[{"op":"remove","path":"/spec/image_digests"}]'

oc delete pod spark-hb-nginx-6db69b6c95-mn88f -n zen
```

**Impact:** 89m of failed upgrade time

#### Recommendations
- Implement automatic image digest removal during upgrades

---

### Issue #2: Db2aaService Operator Not Auto-Upgraded
**GitHub Issue:** [#77756](https://github.ibm.com/PrivateCloud-analytics/CPD-Quality/issues/77756)

**Severity:** 🔴 Critical - Blocked WKC upgrade at 50%

**Component:** Db2aaService  
**Affected CR:** `db2aaservice-cr`  
**Migration Job:** Repeatedly crashing (ExitCode=1)

#### Problem Description
WKC upgrade updated Db2aaserviceService CR to 5.3.0, but the operator only supported 5.1.2/5.1.0, blocking WKC at 50% progress.

#### Root Cause
The Db2aaservice operator was not upgraded before the WKC upgrade attempted to update the Db2aaserviceService CR to version 5.3.0. The operator only supported versions 5.1.2 and 5.1.0, causing a version mismatch that blocked WKC upgrade progress.

#### Workaround Applied
```bash
cpd-cli manage case-download --components=db2aaservice --release=5.3.0

cpd-cli manage install-components --components=db2aaservice --release=5.3.0 \
  --operator_ns=cpd-operators --instance_ns=zen --upgrade=true
```

**Duration:** 11m 52s | **Impact:** Blocked WKC at 50%, undocumented manual step

#### Recommendations
- Implement automatic dependent operator upgrades before updating CR versions

---

### Issue #3: Policy CR Image Digest Not Auto-Removed
**GitHub Issue:** [#77757](https://github.ibm.com/PrivateCloud-analytics/CPD-Quality/issues/77757)

**Severity:** 🔴 Critical - Blocked WKC upgrade at 50%

**Component:** WKC Policy Service  
**Affected Pod:** `wdp-policy-service`  
**Status:** Running but failing readiness/liveness probes (6 restarts over 139 minutes)

#### Problem Description
Policy CR's `wdp_policy_service_image` with pinned 5.1.2 digest was not auto-removed, causing the pod to fail readiness/liveness probes (6 restarts over 139 minutes).

#### Root Cause
The pinned 5.1.2 image digest was not automatically removed during upgrade, preventing the operator from updating to the compatible 5.3.0 image.

#### Workaround Applied
```bash
oc patch policy policy-cr -n zen --type=json \
  --patch '[{"op":"remove","path":"/spec/wdp_policy_service_image"}]'

oc delete pod wdp-policy-service-5f9d4fd7bf-7q779 -n zen

oc delete pod -n cpd-operators -l name=ibm-cpd-wkc-operator
```

**Impact:** WKC stuck at 50%, manual CR patching required

#### Recommendations
- Implement automatic image digest removal during upgrades

---

### Issue #4: Canvas Base Malformed Image Paths
**GitHub Issue:** [#77783](https://github.ibm.com/PrivateCloud-analytics/CPD-Quality/issues/77783)

**Severity:** 🟡 Medium - Caused ImagePullBackOff for multiple pods

**Component:** Canvas Base (SPSS Modeler dependency)
**Affected Pods:** `canvasbase-flow-ui`, `canvasbase-flow-api`

#### Problem Description
Canvas Base deployments had malformed image paths with duplicate `/cp/cpd/` segments, causing ImagePullBackOff.

#### Root Cause
Canvas Base operator generated deployments with incorrect image paths containing duplicate `/cp/cpd/` segments.

#### Workaround Applied
Fixed image paths directly in Canvas Base deployments:

```bash
# Edit flow-api deployment
oc edit deployment canvasbase-flow-api -n zen
# Changed to: cp.icr.io/cp/cpd/flow-api@sha256:4a46d186f452393404bc825cfcb1bacf6db3e90591a2f35c64c9a92bdf37d399

# Edit flow-ui deployment
oc edit deployment canvasbase-flow-ui -n zen
# Changed to: cp.icr.io/cp/cpd/flow-ui@sha256:58564a08699fae5c0d84900713655be7977eae133b722d243d8cbf7a3ddbb867
```

**Result:** New pods rolled out successfully with correct 5.3.0 images:
```
canvasbase-flow-api-df5cb5748-rp89x    1/1   Running   0   8m20s
canvasbase-flow-ui-56d9fc744c-s4fqg    1/1   Running   0   6m50s
```

**Impact:** Extended upgrade time, required manual deployment edits

#### Recommendations
- Update Canvas Base operator to generate correct image paths

---

### Issue #5: DataStage CR Does Not Auto-Upgrade
**Severity:** 🟡 Medium - Required manual CR patch to trigger upgrade

**Component:** DataStage Enterprise  
**Operator Version:** 5.3.0  
**CR Status:** Stuck at 5.1.2 despite operator upgrade

#### Problem Description
DataStage CR did not automatically upgrade from 5.1.2 to 5.3.0 after the DataStage operator was upgraded.

#### Root Cause
Helm chart configuration `datastageEnt.deployCR: false` prevented automatic CR upgrade.

#### Workaround Applied
Manually patched the DataStage CR to trigger upgrade:

```bash
# Verify operator is at 5.3.0
# image: icr.io/cpopen/ds-operator@sha256:3f5b4574ced1b24436efae4f7325230d43640bc67c5a52db3a550cf3a0aa1640
# productVersion: 5.3.0

# Manually trigger CR upgrade via patch
oc patch datastage datastage -n zen --type=merge --patch '{"spec":{"version":"5.3.0"}}'
```

**Result:** DataStage CR successfully upgraded to 5.3.0 after manual patch (completed in ~13 minutes)

```
NAME        VERSION   RECONCILED   STATUS      PERCENT   AGE
datastage   5.3.0     5.3.0        Completed   100%      15d
```

**Impact:** Required manual intervention, undocumented step

#### Recommendations
- Update the Helm chart to set `deployCR: true` during upgrades

---

### Issue #6: DMC Service Instance Does Not Auto-Upgrade
**GitHub Issue:** [#77772](https://github.ibm.com/PrivateCloud-analytics/CPD-Quality/issues/77772)

**Severity:** 🟡 High - Service instance stuck, contradicts documentation

**Component:** Data Management Console (DMC)
**Service Instance ID:** 1770680059489170
**DMC Operator Version:** 5.10.0-101

#### Problem Description
DMC service instance did not auto-upgrade despite documentation. Manual upgrade via `cpd-cli service-instance upgrade` got stuck in `UPGRADE_IN_PROGRESS` indefinitely.

#### Root Cause
The cpd-cli tool set an invalid `scaleConfig: <no value>` value, and service instance tracking became disconnected from CR reconciliation status.

#### Workaround Applied
```bash
oc patch dmc data-management-console -n zen --type=merge \
  -p '{"spec":{"scaleConfig":"small"}}'

cpd-cli service-instance upgrade --service-type=dmc \
  --instance-name=data-management-console --profile=cpadmin
```

**Status:** ⚠️ Partial - CR upgraded but instance stuck in UPGRADE_IN_PROGRESS

#### Recommendations
- Confirm scaleConfig is set to a correctly in dmc CR (small/medium/large)

---

## Component Upgrade Details

### Successfully Upgraded Components

| Component | Version | Duration | Issues | Notes |
|-----------|---------|----------|--------|-------|
| CPD Platform | 5.3.0 | N/A | None | Base platform |
| Scheduler | 5.3.0 | 5m 8s | None | Clean upgrade |
| WKC | 5.3.0 | ~15m (retry) | Issues #1, #2, #3 | Required 3 manual fixes |
| DataStage Enterprise | 5.3.0 | 13m | Canvas operator restart | Minor issue |
| DataStage Enterprise Plus | 5.3.0 | Included | None | Bundled with DS |
| Watson Studio | 5.3.0 | Included | None | Clean upgrade |
| Watson Studio Runtimes | 5.3.0 | Included | None | Clean upgrade |
| Watson Machine Learning | 5.3.0 | Included | None | Clean upgrade |
| Watson Pipelines | 5.3.0 | Included | None | Clean upgrade |
| Data Privacy | 5.3.0 | 18m 35s | None | Clean upgrade |
| Mantaflow | 5.3.0 | ~15m | Transient subscription issue | Required 2 retries |
| AnalyticsEngine | 5.3.0 | Included | Issue #1 | Image digest removal required |
| DMC | 5.3.0 | 30m 22s | Issue #4 | CR upgraded, instance stuck |
| DB2 Warehouse | 5.3.0 | Included | None | Clean upgrade |
| Data Virtualization | 5.3.0 | Included | None | Clean upgrade |
| RStudio | 5.3.0 | 20m | Issue #5 | ImagePullBackOff resolved |
| SPSS | 5.3.0 | Included | None | Clean upgrade |
| Cognos Analytics | 5.3.0 | Included | None | Clean upgrade |
| Planning Analytics | 5.3.0 | Included | None | Clean upgrade |
| Db2aaService | 5.3.0 | 11m 52s | Issue #2 | Manual operator upgrade required |

### Service Instance Upgrades

| Instance Type | Count | Command Duration | Actual Duration | Status | Notes |
|---------------|-------|------------------|-----------------|--------|-------|
| Spark | 1 | 31s | ~31s | ✅ Complete | Instance ID: 1770658860342305 |
| Data Virtualization | 1 | 31s | ~70 min | ✅ Complete | Instance ID: 1771879544156740, migration job took ~1 hour |
| DB2 OLTP | 0 | N/A | N/A | Skipped | No instances to upgrade |
| DB2 Warehouse | 0 | N/A | N/A | Skipped | No instances to upgrade |
| Cognos Analytics | 0 | 1s | 1s | No instances | No instances found |
| DMC | 1 | N/A | N/A | ⚠️ Stuck | Instance ID: 1770680059489170, stuck in UPGRADE_IN_PROGRESS |

---

## Post-Upgrade Migrations

- Catalog-API migration from CouchDB to PostgreSQL completed successfully
- IKC migration from Db2 to PostgreSQL completed successfully

---

### Recommendations for Future Upgrades 🎯

**1. Pre-Upgrade Validation**
- Check for pinned image digests, operator version compatibility

**2. Automation Improvements**
- Implement automated removal of pinned image digests during upgrade process
- Add dependency-aware operator upgrade sequencing (e.g., Db2aaService before WKC)

**3. Documentation & Knowledge Management**
- Maintain comprehensive list of known upgrade issues with tested workarounds

---

## Issue Tracking

| Issue | GitHub | Priority | Status | Impact |
|-------|--------|----------|--------|--------|
| AnalyticsEngine Image Digests | [#77755](https://github.ibm.com/PrivateCloud-analytics/CPD-Quality/issues/77755) | Critical | Open | All 5.1.2 upgrades with hotfixes |
| Db2aaService Operator | [#77756](https://github.ibm.com/PrivateCloud-analytics/CPD-Quality/issues/77756) | Critical | Open | All WKC upgrades with Db2aaService |
| Policy CR Image Digest | [#77757](https://github.ibm.com/PrivateCloud-analytics/CPD-Quality/issues/77757) | Critical | Open | All 5.1.2 upgrades with Policy hotfixes |
| DMC Service Instance | [#77772](https://github.ibm.com/PrivateCloud-analytics/CPD-Quality/issues/77772) | High | Open | All DMC service instances |
| Canvas Base Image Paths | [#77783](https://github.ibm.com/PrivateCloud-analytics/CPD-Quality/issues/77783) | Medium | Resolved | Canvas Base deployments |
| DataStage CR Auto-Upgrade | TBD | Medium | Resolved | DataStage CR upgrades |
