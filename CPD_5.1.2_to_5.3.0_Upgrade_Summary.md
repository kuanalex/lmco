# Cloud Pak for Data Upgrade Summary: 5.1.2 → 5.3.0
Author: Alex Kuan (alex.kuan@ibm.com)

## Executive Summary

Successfully completed upgrade of Cloud Pak for Data from version 5.1.2 to 5.3.0 on LMCO Internal Cluster. The upgrade encountered **5 defects** requiring manual intervention.

**Overall Status:** ✅ Completed with manual workarounds

**Total Components Upgraded:** 20+ components including WKC, DataStage, Watson Studio, WML, DMC, and more

**Total Effort:** ~16 man-hours over 2+ business days (Feb 23-25, 2026)

**Command Execution Time:** ~6.5 hours (actual CLI runtime)

**Troubleshooting & Manual Fixes:** ~9.5 hours (defect investigation, workarounds, retries)

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

#### Root Cause - Chicken-and-Egg Problem
1. WKC upgrade starts and updates Db2aaserviceService CR to version 5.3.0
2. Db2aaservice operator (still at 5.1.2 level) rejects the 5.3.0 version
3. WKC upgrade waits for Db2aaservice to be ready
4. Db2aaservice never becomes ready because operator doesn't support 5.3.0
5. **Deadlock:** WKC can't proceed, Db2aaservice operator can't upgrade automatically

The WKC upgrade process does not automatically trigger the Db2aaservice operator upgrade, creating a dependency deadlock.

#### Workaround Applied
```bash
# Download Db2aaservice case bundles for 5.3.0
cpd-cli manage case-download \
  --components=db2aaservice \
  --release=5.3.0

# Manually upgrade Db2aaservice operator
cpd-cli manage install-components \
  --license_acceptance=true \
  --components=db2aaservice \
  --release=5.3.0 \
  --operator_ns=cpd-operators \
  --instance_ns=zen \
  --image_pull_prefix=icr.io \
  --image_pull_secret=ibm-entitled-regcred \
  --upgrade=true
```

**Duration:** 11m 52s for manual operator upgrade

**Impact:** 
- Blocked WKC upgrade at 50% progress
- Required manual intervention not documented in upgrade procedures
- Affects all customers with Db2aaservice installed who are upgrading WKC

#### Recommendations
- **Automatic Operator Upgrade:** Update WKC upgrade process to automatically upgrade dependent operators (like Db2aaservice) before updating CR versions
- **Operator Upgrade Sequencing:** Ensure Db2aaservice operator is upgraded to support 5.3.0 BEFORE the WKC upgrade updates the CR version
- **Version Compatibility Check:** Add pre-upgrade validation that checks if all dependent operators support the target version
- **Graceful Version Handling:** Update WKC upgrade logic to check operator `supportedOperandVersions` before updating CR versions
- **Documentation:** Document that Db2aaservice operator must be manually upgraded before WKC upgrade (until automatic upgrade is implemented)
- Verify similar issues don't exist with other dependent services (Db2, MongoDB, etc.)

---

### Issue #3: Policy CR Image Digest Not Auto-Removed
**GitHub Issue:** [#77757](https://github.ibm.com/PrivateCloud-analytics/CPD-Quality/issues/77757)

**Severity:** 🔴 Critical - Blocked WKC upgrade at 50%

**Component:** WKC Policy Service  
**Affected Pod:** `wdp-policy-service`  
**Status:** Running but failing readiness/liveness probes (6 restarts over 139 minutes)

#### Problem Description
Policy CR's `wdp_policy_service_image` with pinned 5.1.2 digest was not auto-removed. The 5.1.2 image has a username construction bug (`user=ikcadminuser=ikcadmin`) and is incompatible with 5.3.0 PostgreSQL, causing authentication failures (6 restarts over 139 minutes).

#### Root Cause - Multi-Layered Issue
1. **Image Digest Not Removed:** Policy CR's `wdp_policy_service_image` section is not automatically removed during upgrade
2. **Image Bug:** The pinned 5.1.2 wdp-policy-service image has a bug where it incorrectly constructs the PostgreSQL connection string username (`ikcadminuser=ikcadmin` instead of `ikcadmin`)
3. **Database Incompatibility:** The 5.3.0 PostgreSQL database uses different authentication that is incompatible with the 5.1.2 image
4. **Operator Doesn't Update:** The Policy operator doesn't update the deployment to use the 5.3.0 image because the image digest is pinned in the CR

#### Workaround Applied
```bash
# Remove pinned image digest from Policy CR
oc patch policy policy-cr -n zen --type=json \
  --patch '[{"op":"remove","path":"/spec/wdp_policy_service_image"}]'

# Delete failed pod to trigger recreation with correct 5.3.0 image
oc delete pod wdp-policy-service-5f9d4fd7bf-7q779 -n zen

# Restart WKC operator to trigger reconciliation
oc delete pod -n cpd-operators -l name=ibm-cpd-wkc-operator
```

After applying this patch, the Policy operator reconciled and created a new deployment with the correct 5.3.0 image that is compatible with the 5.3.0 PostgreSQL database.

**Impact:** WKC stuck at 50% progress; required manual CR patching

#### Recommendations
- **Automatic Image Digest Removal:** Update Policy operator upgrade logic to automatically remove `wdp_policy_service_image` section during version upgrades (similar to how `auto_hotfix_removal` should work for AnalyticsEngine)
- **Pre-upgrade Validation:** Add validation that checks for pinned image digests before starting upgrade; warn or automatically remove incompatible image digests
- **Operator Reconciliation Fix:** Ensure Policy operator properly reconciles and updates deployments when image digest section is removed
- **Image Compatibility Check:** Add validation that pinned images are compatible with target database versions; fail fast with clear error message if incompatibility is detected
- **Fix Image Bug:** Review and fix username construction bug in older wdp-policy-service images (if backporting)
- Verify similar issues don't exist in other WKC sub-components (UG, IIS, Glossary, etc.)

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

#### Root Cause - Three Separate Issues
1. **cpd-cli Bug:** The `cpd-cli service-instance upgrade` command set `scaleConfig` to the literal string `<no value>` instead of a valid scale configuration value
2. **Service Instance Disconnect:** After the DMC CR successfully upgrades to 5.3.0, the service instance upgrade process does not complete. There appears to be a disconnect between:
   - DMC CR reconciliation (completed successfully)
   - Service instance upgrade tracking (stuck in UPGRADE_IN_PROGRESS)
3. **No Auto-Upgrade:** Service instances not automatically upgraded when operator/CR upgraded, contradicting IBM documentation

The zen-core-api (which manages service instance upgrades) shows no activity related to DMC, suggesting the service instance upgrade job was never created or failed silently.

#### Workaround Applied
```bash
# Fix scaleConfig in DMC CR
oc patch dmc data-management-console -n zen --type=merge \
  -p '{"spec":{"scaleConfig":"default"}}'

# Manual service instance upgrade (still stuck as of report time)
cpd-cli service-instance upgrade \
  --service-type=dmc \
  --instance-name=data-management-console \
  --profile=cpadmin
```

**Status:** ⚠️ Partially resolved - DMC CR upgraded to 5.3.0 but service instance status still stuck in UPGRADE_IN_PROGRESS

#### Recommendations
- **Fix cpd-cli:** Update `cpd-cli service-instance upgrade` to set a valid default scaleConfig value instead of `<no value>`
- **Fix Service Instance Upgrade:** Investigate why the service instance upgrade process does not complete after the DMC CR upgrade succeeds:
  - Service instance upgrade job creation logic
  - Zen-core-api service instance tracking
  - DMC operator service instance status update mechanism
- **Fix Auto-Upgrade:** Implement automatic service instance upgrade when DMC operator/CR is upgraded, as documented
- **Immediate Workaround Needed:** Manual recovery procedure to complete the stuck service instance upgrade; command or process to reset service instance status and retry upgrade

---

### Issue #5: Missing Images from Production Repository
**Severity:** 🟡 Medium - Caused ImagePullBackOff for multiple pods

**Component:** Canvas Base, RStudio  
**Affected Pods:** `canvasbase-flow-ui`, `canvasbase-flow-api`, RStudio pods

#### Problem Description
Production repository images for Canvas Base and RStudio were missing/inaccessible, causing ImagePullBackOff and extending upgrade time.

#### Workaround Applied
```bash
# Restart Canvas operator pod to retry image pull
oc delete pod ibm-cpd-canvasbase-operator-d85d5498f-s6x95 -n cpd-operators

# Verify new operator pod is running
oc get pod -n cpd-operators | grep canvasbase
```

Some images eventually became available after operator restart, but RStudio ImagePullBackOff issues persisted beyond the upgrade window.

**Impact:** 
- Extended upgrade time for affected components
- Some components not fully operational immediately after upgrade
- Required manual intervention and monitoring

#### Recommendations
- **Pre-Release Validation:** Ensure all required images are available in production repository before release
- **Pre-Upgrade Image Check:** Implement pre-upgrade image availability validation
- **Fallback Mechanisms:** Add fallback mechanisms for missing images
- **Better Error Messages:** Provide clearer error messages indicating which images are missing and from which repository

---

## Common Patterns & Root Causes

### Pattern #1: Image Digest Pinning Issues
**Affected Components:** AnalyticsEngine, Policy CR, (partially) DMC

Three of the five critical issues involved pinned image digests from 5.1.2 hotfixes not being automatically removed during upgrade:

1. **AnalyticsEngine:** `image_digests` section not removed despite `auto_hotfix_removal: true`
2. **Policy CR:** `wdp_policy_service_image` section not removed
3. **DMC:** `scaleConfig` issue related to image/configuration management

**Common Root Cause:**
- Operators lack proper upgrade reconciliation logic to detect and remove pinned image digests
- No pre-upgrade validation to detect these issues before starting upgrade
- Inconsistent implementation of hotfix removal across different operators

**Systemic Impact:**
- Blocks upgrades at various stages (50%, 85%)
- Requires manual CR patching
- Not documented in upgrade procedures
- Affects all customers who applied hotfixes before upgrading

**Recommended Solution:**
Implement comprehensive pre-upgrade validation to detect and remove all pinned image digests before starting upgrade process. Update all operators to properly handle image digest removal during version upgrades.

### Pattern #2: Operator Upgrade Sequencing Problem
**Affected Component:** Db2aaService

Issue #2 (Db2aaService) revealed a fundamental gap in upgrade orchestration:

**The Problem:**
- WKC upgrade updates dependent CR versions before ensuring operators support those versions
- No automatic operator upgrade for dependencies
- Creates deadlock situation requiring manual intervention

**Chicken-and-Egg Sequence:**
1. WKC upgrade starts → updates Db2aaserviceService CR to 5.3.0
2. Db2aaservice operator (still at 5.1.2) → rejects 5.3.0 version
3. WKC upgrade → waits for Db2aaservice to be ready
4. Db2aaservice → never becomes ready (operator doesn't support 5.3.0)
5. **Deadlock:** WKC can't proceed, Db2aaservice operator can't upgrade automatically

**Expected Flow:**
1. WKC upgrade detects dependent operators that need upgrading
2. Automatically upgrade all dependent operators first (including Db2aaservice to support 5.3.0)
3. Then upgrade CRs to new versions
4. This ensures operators are ready to handle new CR versions

**Recommended Solution:**
Implement dependency-aware operator upgrade sequencing in WKC upgrade workflow. Add pre-upgrade validation for operator version compatibility.

### Pattern #3: Service Instance Upgrade Disconnect
**Affected Component:** DMC

Issue #4 revealed problems with service instance upgrade tracking:

**The Problem:**
- Service instances don't automatically upgrade when operator/CR upgraded (contradicts documentation)
- Manual upgrade gets stuck in UPGRADE_IN_PROGRESS indefinitely
- Disconnect between CR reconciliation (successful) and service instance tracking (stuck)

**Recommended Solution:**
- Implement automatic service instance upgrades as documented
- Fix service instance upgrade job creation and tracking logic
- Add timeout and retry logic for stuck upgrades

### Pattern #4: Documentation Gaps
Multiple issues not documented in official upgrade procedures:

- Manual Db2aaService operator upgrade requirement
- Image digest removal steps
- Service instance upgrade behavior discrepancies
- scaleConfig bug in cpd-cli
- Missing production repository images

**Recommended Solution:**
Comprehensive documentation update including:
- Pre-upgrade checklist for operator versions
- Troubleshooting guide for image digest issues
- Service instance upgrade procedures
- Known issues and workarounds

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
| RStudio | 5.3.0 | 20m | Issue #5 | ImagePullBackOff persisted |
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

## Pre-Upgrade Validation Results

✅ **Pre-migration check passed:** "The system is ready for migration. Upgrade your cluster as usual."

**Critical Gap Identified:**
Pre-upgrade validation did NOT detect the issues that would later block the upgrade:
- Pinned image digests in AnalyticsEngine, Policy CR
- Db2aaService operator version incompatibility
- DMC service instance upgrade behavior
- Missing production repository images

**Recommendation:** Enhance pre-upgrade validation to detect these issues before starting upgrade.

---

## Post-Upgrade Tasks

### Completed ✅
- Spark instance upgraded (Instance ID: 1770658860342305)
- Data Virtualization instance upgraded (Instance ID: 1771879544156740)
- Canvas operator restarted
- All component CRs reconciled to 5.3.0

### Pending ⚠️
- **DMC service instance stuck in UPGRADE_IN_PROGRESS** (Instance ID: 1770680059489170)
  - DMC CR successfully upgraded to 5.3.0
  - Service instance status not updating
  - Requires investigation and manual recovery procedure
  
- **DV DIAGPATH cleanup required**
  ```bash
  oc -n zen rsh c-db2u-dv-db2u-0 bash
  # Follow DV post-upgrade cleanup procedures
  ```

- **RStudio ImagePullBackOff resolution**
  - Some RStudio pods still experiencing image pull issues
  - May require production repository image availability

- **Missing production repository images investigation**
  - Canvas Base flow-ui and flow-api images
  - Need to verify image availability and update image references if needed

---

## Recommendations for Future Upgrades

### Immediate Actions (High Priority)

1. **Pre-Upgrade Script Enhancement:**
   - Add validation to detect pinned image digests in all CRs
   - Check operator version compatibility with target release
   - Validate service instance upgrade eligibility
   - Check production repository image availability
   - Fail fast with clear error messages for any incompatibilities

2. **Operator Upgrade Automation:**
   - Implement automatic dependent operator upgrades in WKC workflow
   - Add dependency graph for operator upgrade sequencing
   - Ensure operators are upgraded BEFORE CRs are updated to new versions
   - Add version compatibility checks before CR updates

3. **Documentation Updates:**
   - Document manual Db2aaService operator upgrade requirement (until automated)
   - Add troubleshooting guide for image digest issues
   - Include service instance upgrade behavior clarifications
   - Add known issues section with workarounds
   - Create pre-upgrade checklist for operator versions

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
| Missing Production Images | TBD | Medium | Pending | Canvas Base, RStudio |

---

## Lessons Learned

**What Went Well ✅**
- Pre-upgrade prep and environment setup smooth
- Shared components and scheduler upgraded cleanly
- Most components upgraded successfully after initial fixes
- Service instance upgrades completed quickly
