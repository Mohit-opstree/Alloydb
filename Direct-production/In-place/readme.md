# AlloyDB PostgreSQL 14 → PostgreSQL 18

## Direct Production In-Place Major Version Upgrade Runbook

---

## 1. Objective

This document describes the procedure for performing a **direct in-place major version upgrade of the production AlloyDB cluster from PostgreSQL 14 to PostgreSQL 18**.

### Current State

```text
Production AlloyDB
PostgreSQL 14
```

### Target State

```text
Production AlloyDB
PostgreSQL 18
```

The upgrade will be performed **on the existing production AlloyDB cluster**.

No separate production clone will be used for application validation before the upgrade.

---

# 2. Upgrade Approach

The implementation will follow this flow:

```text
Production AlloyDB
PostgreSQL 14
       |
       | Pre-upgrade checks
       |
       ↓
Production PostgreSQL 14
       |
       | Create additional recovery backup
       |
       ↓
Production PostgreSQL 14
       |
       | Start in-place major version upgrade
       |
       ↓
Production PostgreSQL 18
       |
       | Post-upgrade validation
       |
       ↓
Application Validation
```

The production cluster itself is upgraded in place.

No data migration to a new production cluster is performed as part of the normal upgrade.

---

# 3. Why Direct Production Upgrade?

The direct approach can be selected when the customer does not want a separate clone-based application testing phase.

### Advantages

* No separate validation environment is required.
* No need to create and maintain an additional AlloyDB cluster for pre-production testing.
* No application cutover to a temporary validation cluster is required.
* Existing production cluster configuration and endpoint are retained.
* The implementation is operationally simpler than a separate migration/cutover approach.
* AlloyDB performs the major version upgrade as an in-place operation.
* AlloyDB performs its own upgrade prechecks before proceeding with the upgrade.

### Important Consideration

Skipping the validation clone means that **application-level compatibility is not tested against PostgreSQL 18 before the production upgrade**.

Therefore, application validation becomes an important step immediately after the production upgrade.

---

# 4. Important Limitations and Risks

## 4.1 No Pre-Production Application Validation

Because the upgrade is being performed directly on production, there is no opportunity to validate the complete application workload against PostgreSQL 18 beforehand.

Potential issues may include:

* Application queries behaving differently.
* Database functions/procedures requiring changes.
* Extension compatibility issues.
* Query performance changes.
* Application connection or transaction issues.
* Unexpected behaviour in database-dependent application functionality.

These issues may only become visible after the production upgrade.

---

## 4.2 Same-Cluster Downgrade Is Not Available

Once the production primary has successfully completed the upgrade to PostgreSQL 18, the same AlloyDB cluster cannot simply be downgraded:

```text
PostgreSQL 18
      |
      X
PostgreSQL 14
```

If recovery to PostgreSQL 14 is required after the primary has been upgraded, the previous PostgreSQL 14 state must be restored to a **new AlloyDB cluster** and the application must subsequently be redirected to that recovery cluster.

---

## 4.3 Production Downtime

The upgrade requires a period during which the production primary database is unavailable.

The actual duration depends on factors such as:

* Database size
* Schema size
* Number of databases
* Number of read-pool instances
* Cluster configuration

The application downtime window should therefore be planned and approved before execution.

---

## 4.4 Extension Compatibility

Installed PostgreSQL extensions must be checked before the upgrade.

An unsupported extension/version can cause the upgrade to fail or require remediation.

Check:

```sql
SELECT extname,
       extversion
FROM pg_extension
ORDER BY extname;
```

All required extensions must be compatible with PostgreSQL 18 before proceeding.

---

# 5. Pre-Upgrade Checklist

The following checks must be completed before starting the production upgrade.

---

## 5.1 Verify Cluster Status

```bash
gcloud alloydb clusters describe "${PROD_CLUSTER_ID}" \
  --region="${REGION}" \
  --project="${PROJECT_ID}"
```

Verify that the production cluster is in a healthy/ready state and that no unexpected operation is in progress.

---

## 5.2 Verify Instance Status

```bash
gcloud alloydb instances list \
  --cluster="${PROD_CLUSTER_ID}" \
  --region="${REGION}" \
  --project="${PROJECT_ID}"
```

Verify the primary and applicable read-pool instances.

Do not proceed if the cluster is already undergoing an unexpected operation or is in an unhealthy state.

---

## 5.3 Verify PostgreSQL Version

Connect to the production database:

```sql
SELECT version();
```

and:

```sql
SHOW server_version;
```

Expected:

```text
PostgreSQL 14.x
```

---

## 5.4 Check Installed Extensions

```sql
SELECT extname,
       extversion
FROM pg_extension
ORDER BY extname;
```

Review every installed extension for PostgreSQL 18 compatibility.

Resolve any unsupported extension/version before starting the upgrade.

---

## 5.5 Check Database Connectivity

```sql
SELECT datname,
       datallowconn
FROM pg_database
ORDER BY datname;
```

Verify that required production databases allow connections.

---

## 5.6 Check `template1`

```sql
SELECT datname,
       datistemplate
FROM pg_database
WHERE datname = 'template1';
```

Verify that `template1` is correctly configured.

---

## 5.7 Check Large Object Metadata

```sql
SELECT count(*)
FROM pg_largeobject_metadata;
```

Expected:

```text
0
```

A non-zero result is a blocker for the in-place major version upgrade and must be remediated before proceeding.

---

## 5.8 Check Logical Replication

If logical replication is configured, identify the applicable slots/subscriptions.

For example:

```sql
SELECT slot_name,
       slot_type,
       active
FROM pg_replication_slots;
```

If the cluster is acting as a logical replication source, follow the AlloyDB documented procedure to disable downstream subscriptions and handle the logical replication slots before the upgrade.

Logical replication must be restored/reconfigured after the upgrade where applicable.

---

## 5.9 Check Cross-Region Replication

If cross-region replication/secondary clusters are configured, this must be addressed before the in-place upgrade.

The in-place major version upgrade does not support upgrading a secondary cluster in place.

The applicable replication configuration must therefore be handled according to the AlloyDB upgrade procedure before proceeding.

---

# 6. Create Additional Production Backup

Before starting the production upgrade, create an additional manual backup.

```bash
gcloud alloydb backups create "${BACKUP_ID}" \
  --cluster="${PROD_CLUSTER_ID}" \
  --region="${REGION}" \
  --project="${PROJECT_ID}" \
  --async
```

Monitor the backup operation and confirm that it completes successfully.

### Mandatory Gate

Do **not** start the production PostgreSQL 14 → PostgreSQL 18 upgrade until the backup has completed successfully.

This manual backup provides an additional controlled recovery point.

AlloyDB also creates an automatic pre-upgrade backup as part of the in-place upgrade workflow.

The manual backup does **not** replace AlloyDB's automatic pre-upgrade backup.

---

# 7. Start Production In-Place Upgrade

Once all prechecks are successful and the backup is confirmed, start the upgrade.

```bash
gcloud alloydb clusters upgrade "${PROD_CLUSTER_ID}" \
  --region="${REGION}" \
  --version=POSTGRES_18 \
  --async
```

This initiates the in-place upgrade of the existing production cluster.

The production cluster remains the same cluster; the PostgreSQL major version changes from 14 to 18.

---

# 8. Upgrade Process

Conceptually, the upgrade follows this process:

```text
Upgrade Request
      |
      ↓
AlloyDB Upgrade Prechecks
      |
      ↓
Upgrade Preparation
      |
      ↓
Primary Instance Upgrade
      |
      ↓
Read Pool Upgrade
(if applicable)
      |
      ↓
Cleanup
      |
      ↓
Upgrade Complete
```

During this period, the production application should be treated as being within the approved maintenance window.

---

# 9. Monitor Upgrade Operation

List the upgrade operations:

```bash
gcloud alloydb operations list \
  --cluster="${PROD_CLUSTER_ID}" \
  --region="${REGION}" \
  --filter="metadata.verb:upgrade"
```

Obtain the operation ID and inspect it:

```bash
gcloud alloydb operations describe "<OPERATION_ID>" \
  --region="${REGION}" \
  --project="${PROJECT_ID}"
```

The operation must be monitored until a final status is available.

---

# 10. Upgrade Result Handling

## 10.1 SUCCESS

```text
SUCCESS
```

Meaning:

* Upgrade completed successfully.
* Primary instance has been upgraded.
* Applicable read-pool instances have completed successfully.

Proceed with post-upgrade validation.

---

## 10.2 FAILED

```text
FAILED
```

Meaning:

The upgrade operation failed.

If the failure occurs before the primary instance has been successfully upgraded, AlloyDB can automatically roll back the upgrade operation and keep the production cluster on PostgreSQL 14.

Actions:

1. Inspect the operation details.
2. Review the upgrade/precheck logs.
3. Identify the failure reason.
4. Remediate the identified issue.
5. Confirm the production cluster is healthy.
6. Repeat the upgrade only after the issue has been resolved.

Do not assume that every failure after the primary has already been upgraded will automatically restore PostgreSQL 14.

---

## 10.3 PARTIAL_SUCCESS

```text
PARTIAL_SUCCESS
```

This can occur when the primary instance has been upgraded successfully but one or more read-pool instances could not be upgraded.

In this situation:

```text
Primary
PG18
  |
  +---- Read Pool 1 → PG18
  |
  +---- Read Pool 2 → Upgrade Failed
```

The cluster should not be assumed to have automatically returned to PostgreSQL 14.

Investigate the failed read-pool instance and follow the documented AlloyDB remediation procedure.

---

# 11. Post-Upgrade Validation

After the upgrade completes successfully, perform the following validations.

---

## 11.1 Verify PostgreSQL Version

```sql
SELECT version();
```

```sql
SHOW server_version;
```

Expected:

```text
PostgreSQL 18.x
```

---

## 11.2 Verify AlloyDB Instance Status

```bash
gcloud alloydb instances list \
  --cluster="${PROD_CLUSTER_ID}" \
  --region="${REGION}" \
  --project="${PROJECT_ID}"
```

Verify that the primary and applicable read-pool instances are healthy.

---

## 11.3 Verify Database Connectivity

Connect to all required production databases and confirm:

* Connection succeeds.
* Required schemas are accessible.
* Required tables are accessible.
* Required views are accessible.
* Required functions/procedures are available.
* Required transactions can be executed.

---

## 11.4 Verify Application

The application team should perform functional validation.

At minimum verify:

```text
Application
    ↓
Database Connection
    ↓
Authentication / Critical APIs
    ↓
Read Operations
    ↓
Write Operations
    ↓
Transactions
    ↓
Critical Business Flows
```

Validate the application's critical production workflows.

---

## 11.5 Verify Extensions

```sql
SELECT extname,
       extversion
FROM pg_extension
ORDER BY extname;
```

Review the extension versions after the upgrade.

Where applicable, update database extension catalogs using:

```sql
ALTER EXTENSION EXTENSION_NAME UPDATE;
```

Example:

```sql
ALTER EXTENSION postgis UPDATE;
```

Only execute extension updates that are applicable and supported for the environment.

---

## 11.6 Refresh Database Statistics

Run:

```sql
ANALYZE;
```

This refreshes PostgreSQL statistics after the major version upgrade and helps the query planner make appropriate execution-plan decisions.

---

## 11.7 Monitor Application and Database

After the upgrade, monitor:

* Application error rate
* API failures
* Database connections
* Query latency
* CPU utilization
* Memory utilization
* Database logs
* Slow queries
* Read-pool health
* Application transaction failures

Continue enhanced monitoring during the initial post-upgrade period.

---

# 12. Recovery and Rollback Strategy

There are two different recovery scenarios.

---

## 12.1 Failure Before Primary Upgrade Completes

If the upgrade fails before the primary instance has been upgraded, AlloyDB may automatically roll back the upgrade operation.

Expected state:

```text
Production
PostgreSQL 14
```

After confirming the cluster is healthy:

```text
Investigate failure
       ↓
Fix issue
       ↓
Run prechecks again
       ↓
Retry upgrade
```

---

## 12.2 Primary Successfully Upgraded to PostgreSQL 18

Once the primary has successfully been upgraded to PostgreSQL 18:

```text
PG18 → PG14
```

cannot be performed as a same-cluster downgrade.

If the business requires restoration to PostgreSQL 14, use the pre-upgrade backup.

Recovery flow:

```text
Pre-Upgrade PostgreSQL 14 Backup
              |
              ↓
     Restore NEW AlloyDB Cluster
              |
              ↓
        PostgreSQL 14
              |
              ↓
      Application Validation
              |
              ↓
      Application Cutover
```

---

# 13. Restore Previous PostgreSQL 14 State

Restore the backup into a new AlloyDB cluster:

```bash
gcloud alloydb clusters restore "${RECOVERY_CLUSTER_ID}" \
  --backup="${PRE_UPGRADE_BACKUP_ID}" \
  --region="${REGION}" \
  --project="${PROJECT_ID}"
```

After the recovery cluster is created:

1. Create/configure the required primary instance.
2. Verify PostgreSQL version.
3. Verify database availability.
4. Verify schemas and application objects.
5. Validate application connectivity.
6. Validate critical application workflows.
7. Update the application connection configuration to use the recovery cluster.
8. Confirm application traffic is successfully operating on PostgreSQL 14.

---

# 14. Recovery Decision Flow

```text
             Production PG14
                   |
                   ↓
             Prechecks
                   |
                   ↓
          Additional Backup
                   |
                   ↓
           Start PG14 → PG18
                   |
             ┌─────┴─────┐
             |           |
          Failure      Success
             |           |
             ↓           ↓
       Before primary   PG18
          upgrade         |
             |            ↓
       AlloyDB may     Validate
       auto-rollback      |
             |       ┌────┴────┐
             |       |         |
             |     Healthy   Critical
             |       |       issue
             |       ↓         |
             |     Stay PG18   ↓
             |             Restore
             |             PG14 backup
             |                  |
             |                  ↓
             |             NEW cluster
             |                  |
             |                  ↓
             |             App cutover
             ↓
          PG14
```

---

# 15. Advantages of Direct Production Upgrade

| Area                                           | Direct In-Place Upgrade             |
| ---------------------------------------------- | ----------------------------------- |
| Additional clone required                      | No                                  |
| Application pre-testing                        | No                                  |
| Data migration                                 | No                                  |
| Production cluster                             | Same cluster                        |
| Application endpoint change for normal upgrade | No                                  |
| Operational complexity                         | Lower                               |
| Additional infrastructure                      | Minimal                             |
| Pre-production application validation          | Not available                       |
| Production compatibility risk                  | Higher than a tested clone approach |
| Same-cluster PG18 → PG14 downgrade             | Not available                       |

---

# 16. Key Limitations

The following limitations must be explicitly acknowledged before approval:

1. Application compatibility with PostgreSQL 18 is not validated beforehand.
2. Extension compatibility must be confirmed before execution.
3. Production downtime is required during the upgrade.
4. A successful primary upgrade cannot simply be reversed to PostgreSQL 14 on the same cluster.
5. Recovery to PostgreSQL 14 after a completed primary upgrade requires restoring the previous state to a new AlloyDB cluster.
6. Read-pool upgrade failures may result in `PARTIAL_SUCCESS`.
7. Logical replication and cross-region replication require additional handling where applicable.
8. Post-upgrade application and database validation is mandatory.

---

# 17. Go / No-Go Criteria

## GO

Proceed with the upgrade only when:

* Production cluster is healthy.
* Production instances are healthy.
* PostgreSQL version is confirmed as 14.x.
* Required extensions are compatible with PostgreSQL 18.
* `pg_largeobject_metadata` check returns `0`.
* Applicable replication requirements have been addressed.
* Required databases allow connections.
* Production backup has completed successfully.
* Maintenance window has been approved.
* Application and database teams are available for post-upgrade validation.
* Recovery plan has been reviewed and accepted.

## NO-GO

Do not start the upgrade if:

* Production cluster is unhealthy.
* An unexpected operation is already running.
* Required extensions are incompatible.
* `pg_largeobject_metadata` contains entries.
* Required replication configuration has not been handled.
* Backup has not completed successfully.
* Application owners are unavailable for validation.
* Recovery/cutover plan has not been approved.

---

# 18. Final Implementation Flow

```text
                    PRODUCTION
                  PostgreSQL 14
                         |
                         ↓
                  Pre-Upgrade Checks
                         |
                         ↓
               Additional Backup Created
                         |
                         ↓
              Backup Completion Confirmed
                         |
                         ↓
             Start In-Place Upgrade
                         |
                         ↓
                  AlloyDB Prechecks
                         |
                         ↓
                  Primary Upgrade
                         |
                         ↓
                Read Pool Upgrade
                  (if applicable)
                         |
                         ↓
                 Upgrade Completed
                         |
                         ↓
              PostgreSQL 18 Validation
                         |
                         ↓
               Application Validation
                         |
                ┌────────┴────────┐
                |                 |
             Healthy          Critical Issue
                |                 |
                ↓                 ↓
            Stay PG18       Restore PG14 Backup
                                  |
                                  ↓
                           New AlloyDB Cluster
                                  |
                                  ↓
                           Application Validation
                                  |
                                  ↓
                           Application Cutover
```

---

# 19. Final Outcome

The planned implementation is:

```text
Production AlloyDB PostgreSQL 14
              |
              | Direct In-Place Upgrade
              ↓
Production AlloyDB PostgreSQL 18
```

The upgrade does not involve migrating production data to a new cluster during the normal upgrade path.

The key operational controls are:

```text
Prechecks
   +
Additional Backup
   +
AlloyDB Automatic Pre-Upgrade Backup
   +
Controlled Maintenance Window
   +
Upgrade Monitoring
   +
Post-Upgrade Application Validation
   +
New-Cluster Recovery Plan
```

This approach provides a controlled direct production upgrade while clearly defining the limitations of skipping pre-production application testing and the recovery procedure if PostgreSQL 14 must be restored.
