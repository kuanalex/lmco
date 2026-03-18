# LMCO Internal Cluster Upgrade Summary: 5.1.2 to 5.3.1

## Executive Summary

Upgrade from version 5.1.2 to 5.3.1 on LMCO Internal Cluster is **fully complete** with 20 of 20 components successfully upgraded. The upgrade encountered **5 issues** requiring manual intervention during the process, all of which have been resolved.

- **Overall status:** All components upgraded successfully (100%)
- **Total components upgraded:** 20 of 20 components - cpd_platform, scheduler, db2aaservice, db2oltp, db2wh, dv, dmc, wkc, dp, mantaflow, datastage_ent, datastage_ent_plus, ws, ws_runtimes, wml, analyticsengine, cognos_analytics, planning_analytics, spss, rstudio
- **Total effort:** 13 hours over 2 business days (March 13 & 16, 2026)
  - **Day 1 (March 13):** 8.5 hours
  - **Day 2 (March 16):** 4.5 hours

## Upgrade Timeline & Duration

### Day 1 - March 13, 2026 (8.5 hours)

| Phase | Duration | Notes |
|-------|----------|-------|
| **Pre-upgrade Setup** | 1h 20m | Client workstation, environment variables, pre-checks, licensing, scheduler, entitlements, cluster scoped resources |
| **Platform Operators Install** | 86m 40s | ZenService does not auto-upgrade |
| **ZenService Investigation and Manual Patch** | 2h | Investigate ZenService failure and manually patched version to 6.4.0 |
| **Image Digest Removal** | 2h 10m | Removed digests from CCS, AE, WKC, Policy |
| **Db2aaService Operator Upgrade** | 2m 8s | Manual workaround required |
| **WKC/DP/Mantaflow** | 60m | Failed initially, completed after additional image digest fixes in WKC sub-CRs |
| **End of Day 1** | - | Paused upgrade; remaining components scheduled for Day 2 |

### Day 2 - March 16, 2026 (4.5 hours)

| Phase | Duration | Notes |
|-------|----------|-------|
| **Db2OLTP/Db2WH/DV/DMC** | 21m 49s | DV failed due to image_digests; Db2OLTP completed successfully |
| **Image Digest Removal (DV/WS/WML)** | 30m | Patched DV, WS, and WML CRs |
| **WML Completion** | 40m | WML reconciled after patch |
| **DataStage/WS/WML/WSPipelines** | ~40m | WML completed after image digest removal; WSPipelines blocked by Tekton issue |
| **RStudio/SPSS/Cognos/Planning Analytics** | ~30m | All analytics components completed successfully |
| **Post-Upgrade Migrations** | 30m | CCS and IKC migrations |
| **Service Instance Fixes** | 1h | DMC instance fixed; DV instance remains in progress |
| **End of Day 2** | - | Upgrade complete with 2 outstanding issues |

## Issues Encountered

### Resolved Issues

| Issue | Impact | Resolution |
|-------|--------|------------|
| ZenService auto-upgrade failure | 3h | Manual patch to 6.4.0 required |
| Pinned image digests (11 CRs: WKC + 8 sub-CRs, DV, WS, WML) | 3.5h | Remove image_digests from all affected CRs |
| DMC service instance stuck | 1h | Fix scaleConfig + restart controllers |

**Key Takeaways:**
- Verify ZenService CR version post-platform install and manually patch if needed
- Pre-emptively remove `image_digests` from WKC, DV, WS, and WML CRs before upgrade

---

### Previously Outstanding, Now Resolved Issues

#### WSPipelines Blocked by Tekton Controller Crash
**Component:** WSPipelines | **GitHub:** [#80026](https://github.ibm.com/PrivateCloud-analytics/CPD-Quality/issues/80026)

`tekton-extensions-controller` crashes attempting to disable locked `APIPriorityAndFairness` feature gate. WSPipelines CR stuck at 5.1.2 (55% progress).

**Status:** ✅ **RESOLVED**

**Root Cause:** Image mismatch in tekton-extensions-controller deployment. The failing pod was referencing a different image than the working pods:
- **Failing pod** (`tekton-extensions-controller-575745f88d-2n9n4`): `cp.icr.io/cp/cpd/tekton-extensions-controller@sha256:fe58dfa174c818dc9f39964dd7b23feeb98bc1fa91f33937c2e3dbac37a18c20`
- **Working pods** (`tekton-extensions-controller-5c7449f45c-2k7tc`): `cp.icr.io/cp/cpd/tekton-extensions-controller@sha256:e440311016dc322342c8919f462e8051a9ce639117ab259604f3efa447946704`

**Resolution:** Updated the tekton deployment to reference the working image (`sha256:e440311016dc322342c8919f462e8051a9ce639117ab259604f3efa447946704`). After the update:
- Tekton controller pod issue resolved
- WSPipelines completed reconcile at 5.1.2
- Successfully proceeded with 5.3.1 upgrade

**Verification:**
```
oc get wspipelines
NAME             VERSION   RECONCILED   STATUS      PERCENT   AGE
wspipelines-cr   5.3.1     5.3.1        Completed   100%      5d16h
```

---

#### DV Service Instance Upgrade Incomplete
**Component:** Data Virtualization | **GitHub:** [#80486](https://github.ibm.com/PrivateCloud-analytics/CPD-Quality/issues/80486)

DV service instance remains at 3.1.2 despite successful CR upgrade to 3.3.1. Upgrade option visible but not executing.

**Status:** ✅ **RESOLVED**

**Root Cause:** Timing issue during upgrade process where DV head pod creation was delayed 2.5 hours after DV hurricane pod started:
- **03/16, 16:49:02 UTC** - Upgrade triggered
- **03/16, 17:01:31 UTC** - Hurricane pod created
- **03/16, 19:47:29 UTC** - DV head pod created (2.5 hour delay due to k8s/OpenShift system issue)

Due to the delay, the hurricane pod downloaded incorrect Hadoop configuration from the head pod, causing the BigSQL scheduler component to fail startup. The downloaded config mismatched the binaries in the DV hurricane pod.

**Resolution Steps:**
1. **Manual Hadoop config download:** From DV hurricane pod (`c-db2u-dv-hurrian`), ran `/var/ibm/bigsql/logs/a.sh` to manually download correct Hadoop config from head pod
2. **SSL configuration:** Set `db2comm` back to `TCPIP,SSL` to enable DB2 SSL port for metastore connection (metastore runs in hurricane pod)
3. **Metastore schema mismatch:** Encountered expected metastore error due to database schema name mismatch (`HIVE` vs `SYSHIVE` in `hive-site.xml`). Confirmed with BigSQL team this is expected during upgrade and would be resolved after BigSQL database schema update
4. **StatefulSet restart:** Scaled down and up DV head/worker StatefulSet:
   ```bash
   oc scale sts c-db2u-dv-db2u --replicas=0
   # Wait for all c-db2u-dv-db2u-xxx pods to be deleted
   oc scale sts c-db2u-dv-db2u --replicas=2
   ```

**Outcome:** DV instance upgrade resumed and completed successfully after StatefulSet restart.
