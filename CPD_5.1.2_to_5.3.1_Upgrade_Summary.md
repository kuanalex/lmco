# LMCO Internal Cluster Upgrade Summary: 5.1.2 to 5.3.1

## Executive Summary

Upgrade from version 5.1.2 to 5.3.1 on LMCO Internal Cluster is substantially complete with 19 of 20 components successfully upgraded. The upgrade encountered **3 issues** requiring manual intervention during the process, with **2 defects** blocking full completion.

- **Overall status:** 19 of 20 components upgraded - 2 defects under investigation
- **Total components upgraded:** 19 of 20 components (95%) - cpd_platform, scheduler, db2aaservice, db2oltp, db2wh, dv, dmc, wkc, dp, mantaflow, datastage_ent, datastage_ent_plus, ws, ws_runtimes, wml, analyticsengine, cognos_analytics, planning_analytics, spss, rstudio
- **Total effort:** 13 hours over 2 business days (March 13 & 16, 2026)
  - **Day 1 (March 13):** 8.5 hours
  - **Day 2 (March 16):** 4.5 hours
- **Pending items:** 2 defects under investigation with development teams (WSPipelines CR stuck at 55%, DV service instance not upgrading)

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

### Outstanding Issues

#### WSPipelines Blocked by Tekton Controller Crash
**Component:** WSPipelines | **GitHub:** [#80026](https://github.ibm.com/PrivateCloud-analytics/CPD-Quality/issues/80026)

`tekton-extensions-controller` crashes attempting to disable locked `APIPriorityAndFairness` feature gate. WSPipelines CR stuck at 5.1.2 (55% progress).

**Status:** Pending Pipelines team resolution.

---

#### DV Service Instance Upgrade Incomplete
**Component:** Data Virtualization

DV service instance remains at 3.1.2 despite successful CR upgrade to 3.3.1. Upgrade option visible but not executing.

**Status:** Pending Data Virtualization team resolution.