# Cloud Pak for Data Upgrade Summary: 5.1.2 → 5.3.0

## Executive Summary

Successfully completed upgrade of Cloud Pak for Data from version 5.1.2 to 5.3.0 on LMCO Internal Cluster. The upgrade encountered **5 defects** requiring manual intervention.

**Overall Status:** ✅ Completed with manual workarounds

**Total Components Upgraded:** 20+ components including WKC, DataStage, Watson Studio, WML, DMC, and more

**Total Effort:** ~16 man-hours over 2+ business days (Feb 23-25, 2026)

**Command Execution Time:** ~6.5 hours (actual CLI runtime)

**Troubleshooting & Manual Fixes:** ~9.5 hours (defect investigation, workarounds, retries)

---

## Environment Details

**Cluster Information:**
- **OCP Cluster:** hukfusion.cp.fyre.ibm.com
- **OCP Console:** https://api.hukfusion.cp.fyre.ibm.com:6443
- **OpenShift Version:** 4.17.47
- **Kubernetes Version:** v1.30.14
- **Platform:** x86_64 (amd64)

**CPD Tools:**
- **CPD CLI Version:** 14.3.0 (Build 2819)
- **CPD CLI Build Date:** 2025-12-10T14:05:48
- **SWH Release Version:** 5.3.0
- **Helm Version:** v4.1.1
- **OC Client Version:** 4.18.28

**Storage Classes:**
- **Block Storage:** `ontap-nas`
- **File Storage:** `ontap-nas`

**Namespaces:**
- **Operators:** `cpd-operators`
- **Operands:** `zen`
- **Cert Manager:** `ibm-cert-manager`
- **Licensing:** `ibm-licensing`
- **Scheduler:** `cpd-scheduler`

**Image Registry:**
- **Pull Prefix:** `icr.io`
- **Pull Secret:** `ibm-entitled-regcred`

---

## Upgrade Timeline & Duration

### Day 1 - February 23, 2026 (~3 hours)

| Phase | Duration | Status | Notes |
|-------|----------|--------|-------|
| **Pre-upgrade Setup** | ~30 min | ✅ Complete | Client workstation, environment variables, pre-checks |
| **Shared Cluster Components** | 3m 26s | ✅ Complete | Timing issue on first attempt, retry successful |
| **Scheduler Upgrade** | 5m 8s | ✅ Complete | Clean upgrade |
| **CASE Package Download** | 5m | ✅ Complete | All component case bundles |
| **Platform Operators Install** | 85m 8s | ✅ Complete | Initial platform and operators |

### Day 2 - February 24, 2026 (~10 hours)

| Phase | Duration | Status | Issues |
|-------|----------|--------|--------|
| **WKC/DP/Mantaflow Attempt 1** | 89m 40s | ❌ Failed | AE, Db2aaService, Policy issues |
| **Issue Investigation & Fixes** | ~2 hours | ⚠️ Manual | Image digest removal, operator upgrade |
| **Db2aaService Operator Upgrade** | 11m 52s | ✅ Complete | Manual workaround required |
| **WKC/DP/Mantaflow Retry** | 121m 40s | ✅ Complete | After all fixes applied |
| **DP Standalone** | 18m 35s | ✅ Complete | Clean upgrade |
| **DMC/DB2WH/DV** | 30m 22s | ✅ Complete | Clean upgrade |
| **DataStage/WS/WML/Pipelines** | 20m | ✅ Complete | Canvas operator restart needed |

### Day 3 - February 25, 2026 (~3 hours)

| Phase | Duration | Status | Notes |
|-------|----------|--------|-------|
| **Service Instance Upgrades** | 31s each | ✅ Complete | Spark, DV instances |
| **DMC Instance Investigation** | ~2 hours | ⚠️ Ongoing | Instance stuck in UPGRADE_IN_PROGRESS |
| **Documentation & Reporting** | ~1 hour | ✅ Complete | Defect logging, summary creation |

---

## Critical Issues Encountered

### Issue #1: AnalyticsEngine Image Digests Not Auto-Removed
**GitHub Issue:** [#77755](https://github.ibm.com/PrivateCloud-analytics/CPD-Quality/issues/77755)

**Severity:** 🔴 Critical - Blocked WKC upgrade at 85%

**Component:** AnalyticsEngine (AE)  
**Affected Pod:** `spark-hb-nginx`  
**Status:** CrashLoopBackOff (16 restarts over 64 minutes)

#### Problem Description
AnalyticsEngine CR upgrade failed despite `auto_hotfix_removal: true`. The `image_digests` section with 5.1.2 digests remained, causing spark-hb-nginx pod to crash (16 restarts over 64 minutes).

#### Root Cause
The operator's `auto_hotfix_removal` logic failed to remove pinned image digests during upgrade. The 5.1.2 images are incompatible with 5.3.0 infrastructure.

#### Workaround Applied
```bash
oc patch ae analyticsengine-sample -n zen --type=json \
  --patch '[{"op":"remove","path":"/spec/image_digests"}]'
oc delete pod spark-hb-nginx-6db69b6c95-mn88f -n zen
```

**Impact:** 89m of failed upgrade time

#### Recommendations
- Fix operator's `auto_hotfix_removal` logic to remove image_digests during upgrades
- Add pre-upgrade validation for pinned image digests
- Verify similar issues in other CRs (Policy, UG, IIS)

---

### Issue #2: Db2aaService Operator Not Auto-Upgraded
**GitHub Issue:** [#77756](https://github.ibm.com/PrivateCloud-analytics/CPD-Quality/issues/77756)

**Severity:** 🔴 Critical - Blocked WKC upgrade at 50%

**Component:** Db2aaService  
**Affected CR:** `db2aaservice-cr`  
**Migration Job:** Repeatedly crashing (ExitCode=1)

#### Problem Description
WKC upgrade updated Db2aaserviceService CR to 5.3.0, but the operator only supported 5.1.2/5.1.0, creating a version mismatch that blocked WKC at 50% progress.

#### Root Cause
WKC upgrade creates a chicken-and-egg problem: it updates the CR to 5.3.0 before ensuring the operator supports that version, creating a deadlock where WKC waits for Db2aaservice, but the operator can't reconcile the unsupported version.

#### Workaround Applied
```bash
cpd-cli manage case-download --components=db2aaservice --release=5.3.0
cpd-cli manage install-components --components=db2aaservice --release=5.3.0 \
  --operator_ns=cpd-operators --instance_ns=zen --upgrade=true
```

**Duration:** 11m 52s | **Impact:** Blocked WKC at 50%, undocumented manual step

#### Recommendations
- Implement automatic dependent operator upgrades before updating CR versions
- Add pre-upgrade validation for operator version compatibility
- Update WKC to check `supportedOperandVersions` before updating CRs
- Document manual workaround until automatic upgrade implemented

---

### Issue #3: Policy CR Image Digest Not Auto-Removed
**GitHub Issue:** [#77757](https://github.ibm.com/PrivateCloud-analytics/CPD-Quality/issues/77757)

**Severity:** 🔴 Critical - Blocked WKC upgrade at 50%

**Component:** WKC Policy Service  
**Affected Pod:** `wdp-policy-service`  
**Status:** Running but failing readiness/liveness probes (6 restarts over 139 minutes)

#### Problem Description
Policy CR's `wdp_policy_service_image` with pinned 5.1.2 digest was not auto-removed. The 5.1.2 image has a username construction bug (`user=ikcadminuser=ikcadmin`) and is incompatible with 5.3.0 PostgreSQL, causing authentication failures (6 restarts over 139 minutes).

#### Root Cause
Multi-layered: (1) Image digest not auto-removed, (2) 5.1.2 image has username construction bug, (3) 5.3.0 PostgreSQL incompatible with old image, (4) Operator won't update deployment while digest is pinned.

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
- Add pre-upgrade validation for pinned image digests
- Ensure operator reconciles properly when digest removed
- Verify similar issues in other WKC sub-components

---

### Issue #4: DMC Service Instance Does Not Auto-Upgrade
**GitHub Issue:** [#77772](https://github.ibm.com/PrivateCloud-analytics/CPD-Quality/issues/77772)

**Severity:** 🟡 High - Service instance stuck, contradicts documentation

**Component:** Data Management Console (DMC)  
**Service Instance ID:** 1770680059489170  
**DMC Operator Version:** 5.10.0-101

#### Problem Description
DMC service instance did not auto-upgrade despite documentation. Manual upgrade via `cpd-cli service-instance upgrade` got stuck in `UPGRADE_IN_PROGRESS` indefinitely, even after DMC CR successfully upgraded to 5.3.0.

**Key Issues:**
1. **No Auto-Upgrade:** Service instance remained at 5.1.2 after operator/CR upgraded
2. **scaleConfig Bug:** cpd-cli set `scaleConfig: <no value>` (literal string) instead of valid value
3. **Stuck Status:** After CR completed (5.3.0, status: Completed), service instance remained stuck with no upgrade activity

#### Root Cause
Three issues: (1) cpd-cli bug sets invalid `scaleConfig: <no value>`, (2) Disconnect between CR reconciliation (successful) and service instance tracking (stuck), (3) No auto-upgrade despite documentation.

#### Workaround Applied
```bash
oc patch dmc data-management-console -n zen --type=merge \
  -p '{"spec":{"scaleConfig":"default"}}'
cpd-cli service-instance upgrade --service-type=dmc \
  --instance-name=data-management-console --profile=cpadmin
```

**Status:** ⚠️ Partial - CR upgraded but instance stuck in UPGRADE_IN_PROGRESS

#### Recommendations
- Fix cpd-cli to set valid default scaleConfig
- Investigate service instance upgrade job creation and tracking
- Implement auto-upgrade as documented
- Provide manual recovery procedure for stuck upgrades

---

### Issue #5: Canvas Base Malformed Image Paths
**Severity:** 🟡 Medium - Caused ImagePullBackOff for multiple pods

**Component:** Canvas Base, RStudio  
**Affected Pods:** `canvasbase-flow-ui`, `canvasbase-flow-api`, RStudio pods

#### Problem Description
Canvas Base deployments had malformed image paths with duplicate `/cp/cpd/` segments, causing ImagePullBackOff. RStudio also experienced image pull issues.

**Root Cause:** Canvas Base operator generated deployments with incorrect image paths:
```
❌ cp.icr.io/cp/cpd/cp/cpd/flow-api@sha256:...  (duplicate /cp/cpd/)
❌ cp.icr.io/cp/cpd/cp/cpd/flow-ui@sha256:...   (duplicate /cp/cpd/)
```

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
- Fix Canvas Base operator to generate correct image paths without duplicate segments
- Add pre-upgrade validation to detect malformed image paths
- Implement automated tests for image path generation
- Improve error messages to clearly indicate malformed paths vs missing images

---

## Common Patterns & Root Causes

### Pattern #1: Image Digest Pinning Issues
**Affected:** AnalyticsEngine, Policy CR, DMC

Three issues involved pinned 5.1.2 image digests not auto-removed during upgrade, blocking at 50-85% progress. Root cause: operators lack proper upgrade reconciliation logic and no pre-upgrade validation exists.

**Solution:** Implement pre-upgrade validation to detect/remove pinned digests; update all operators to handle digest removal during upgrades.

### Pattern #2: Operator Upgrade Sequencing
**Affected:** Db2aaService

WKC updates dependent CR versions before ensuring operators support them, creating deadlock. Expected: auto-upgrade dependent operators first, then update CRs.

**Solution:** Implement dependency-aware operator upgrade sequencing; add pre-upgrade validation for operator compatibility.

### Pattern #3: Service Instance Upgrade Disconnect
**Affected:** DMC

Service instances don't auto-upgrade (contradicts docs); manual upgrade gets stuck. Disconnect between CR reconciliation (successful) and instance tracking (stuck).

**Solution:** Implement auto-upgrade as documented; fix job creation/tracking; add timeout/retry logic.

### Pattern #4: Documentation Gaps
Multiple undocumented issues: manual Db2aaService upgrade, image digest removal, service instance behavior, cpd-cli bugs, malformed image paths.

**Solution:** Update docs with pre-upgrade checklist, troubleshooting guide, known issues/workarounds.

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

| Instance Type | Count | Duration | Status | Notes |
|---------------|-------|----------|--------|-------|
| Spark | 1 | 31s | ✅ Complete | Instance ID: 1770658860342305 |
| Data Virtualization | 1 | 31s | ✅ Complete | Instance ID: 1771879544156740 |
| DB2 OLTP | 0 | N/A | Skipped | No instances to upgrade |
| DB2 Warehouse | 0 | N/A | Skipped | No instances to upgrade |
| Cognos Analytics | 0 | 1s | No instances | No instances found |
| DMC | 1 | N/A | ⚠️ Stuck | Instance ID: 1770680059489170, stuck in UPGRADE_IN_PROGRESS |

---

## Post-Upgrade Tasks

### Completed ✅
- Service instances upgraded (Spark: 1770658860342305, DV: 1771879544156740)
- All component CRs reconciled to 5.3.0
- Canvas Base image paths fixed (flow-ui and flow-api deployments with malformed paths)
- DV DIAGPATH cleanup completed

### Pending ⚠️
- **DMC service instance** (ID: 1770680059489170) - Stuck in UPGRADE_IN_PROGRESS
  - DMC CR successfully upgraded to 5.3.0
  - Service instance status not updating
  - Requires manual recovery procedure or IBM support investigation

---

## Recommendations for Future Upgrades

### Immediate (High Priority)
1. **Pre-Upgrade Validation:** Detect pinned digests, operator compatibility, service instance eligibility, image availability
2. **Operator Automation:** Auto-upgrade dependent operators before updating CRs; add dependency sequencing
3. **Documentation:** Add troubleshooting guide, known issues, pre-upgrade checklist

### Long-Term (Medium Priority)
4. **Auto Hotfix Removal:** Fix operators to remove digests; standardize across all components; add automated tests
5. **Service Instance Upgrade:** Fix tracking; implement auto-upgrade; add timeout/retry logic
6. **Image Management:** Ensure availability before release; add validation; improve error messages

### Testing (Ongoing)
7. **Automated Tests:** Test digest removal, version compatibility, service instance workflows across upgrade paths
8. **Cross-Component:** Verify similar issues don't exist in other CRs (UG, IIS, Glossary, Db2, MongoDB)

---

## Defect Tracking

| Issue | GitHub | Priority | Status | Impact |
|-------|--------|----------|--------|--------|
| AnalyticsEngine Image Digests | [#77755](https://github.ibm.com/PrivateCloud-analytics/CPD-Quality/issues/77755) | Critical | Open | All 5.1.2 upgrades with hotfixes |
| Db2aaService Operator | [#77756](https://github.ibm.com/PrivateCloud-analytics/CPD-Quality/issues/77756) | Critical | Open | All WKC upgrades with Db2aaService |
| Policy CR Image Digest | [#77757](https://github.ibm.com/PrivateCloud-analytics/CPD-Quality/issues/77757) | Critical | Open | All 5.1.2 upgrades with Policy hotfixes |
| DMC Service Instance | [#77772](https://github.ibm.com/PrivateCloud-analytics/CPD-Quality/issues/77772) | High | Open | All DMC service instances |
| Canvas Base Image Paths | TBD | Medium | Resolved | Canvas Base deployments |

---

## Lessons Learned

**What Went Well ✅**
- Pre-upgrade prep and environment setup smooth
- Shared components and scheduler upgraded cleanly
- Most components upgraded successfully after initial fixes
- Service instance upgrades completed quickly
