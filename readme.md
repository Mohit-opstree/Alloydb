# AlloyDB In-Place Major Version Upgrade Implementation Guide
## PostgreSQL 14 → PostgreSQL 18

---

## 1. Objective

Exactly what is being implemented:

* **AlloyDB PostgreSQL 14 → PostgreSQL 18**
* **In-place major version upgrade**


---

## 2. Current → Target Architecture

### Architecture Flow

```
CURRENT
Production AlloyDB Cluster
        |
        └── PostgreSQL 14
              |
              | PITR
              v
VALIDATION CLONE
        |
        └── PostgreSQL 14
              |
              | In-place upgrade
              v
        PostgreSQL 18
              |
              └── Application validation

Then:

PRODUCTION
Production AlloyDB Cluster
        |
        └── PostgreSQL 14
              |
              | Pre-upgrade backup
              v
        In-place upgrade
              |
              v
        PostgreSQL 18
              |
              └── Application validation
```

### Important Architecture Note
The validation clone is only used to verify the PostgreSQL 18 upgrade. It is not migrated or copied back into production. The production cluster is independently upgraded in-place.

---

## 3. Phase 1 — Production Pre-Checks

Execute these validation checks on the existing production cluster before initiating any maintenance actions.

### 3.1 Cluster & Instance Status
Check cluster and instance states using the Google Cloud CLI:

```bash
gcloud alloydb clusters describe "${PROD_CLUSTER_ID}" \
  --region="${REGION}" \
  --project="${PROJECT_ID}"
```

```bash
gcloud alloydb instances list \
  --cluster="${PROD_CLUSTER_ID}" \
  --region="${REGION}" \
  --project="${PROJECT_ID}"
```

* **Acceptable:** Cluster state is `READY` and instance state is `READY`.
* **Blocker:** Any cluster or instance in `FAILED`, `MAINTENANCE`, or unexpected mutating state.

---

### 3.2 PostgreSQL Version
```sql
SELECT version();
SHOW server_version;
```
* **Acceptable:** Returns PostgreSQL version `14.x`.
* **Blocker:** Any unexpected version mismatch or unavailable server response.

---

### 3.3 Extensions
```sql
SELECT extname, extversion 
FROM pg_extension 
ORDER BY extname;
```
* **Acceptable:** All installed extensions are standard and compatible with PostgreSQL 18.
* **Blocker:** Deprecated or unsupported third-party extensions that fail to compile or load in PG18.

---



### 3.4 Replication Configuration  (IF APPLICABLE )
* **Logical Replication:** Verify replication slots and active subscriptions:
  ```sql
  SELECT slot_name, plugin, active FROM pg_replication_slots;
  ```
  * **Acceptable:** Inactive replication during maintenance window or slots prepared for cutover.
  * **Blocker:** Active transactions holding slots open preventing database stop.
* **Cross-Region Replication:**
  * **Acceptable:** Cluster is an independent primary or primary in a supported configuration.
  * **Blocker:** Upgrading a secondary replica cluster directly in a cross-region replication pair before promoting or detaching.

---

## 4. Phase 2 — Create PITR Validation Clone

### Exact Command to Restore Clone:
```bash
gcloud alloydb clusters restore "${CLONE_CLUSTER_ID}" \
  --source-cluster="${PROD_CLUSTER_ID}" \
  --point-in-time="${PITR_TIMESTAMP}" \
  --region="${REGION}" \
  --project="${PROJECT_ID}"
```

### Internal Flow:
```
Production Cluster PG14
        |
        | PITR restore
        v
New AlloyDB Clone Cluster
        |
        └── Data restored to selected timestamp
```

### Exact Command to Create the Primary Instance:
```bash
gcloud alloydb instances create "${CLONE_INSTANCE_ID}" \
  --instance-type=PRIMARY \
  --cpu-count=<CPU_COUNT> \
  --region="${REGION}" \
  --cluster="${CLONE_CLUSTER_ID}" \
  --project="${PROJECT_ID}"
```

---

## 5. Phase 3 — Upgrade Clone PG14 → PG18

### Exact Upgrade Command:
```bash
gcloud alloydb clusters upgrade "${CLONE_CLUSTER_ID}" \
  --region="${REGION}" \
  --version=POSTGRES_18 \
  --async
```

### Actual Upgrade Flow:
```
Upgrade request
      ↓
AlloyDB pre-upgrade checks
      ↓
PostgreSQL upgrade preparation
      ↓
Primary instance upgrade
      ↓
Read-pool upgrade (if applicable)
      ↓
Cleanup
      ↓
Upgrade completed
```

### Monitoring the Upgrade:
```bash
gcloud alloydb operations list \
  --cluster="${CLONE_CLUSTER_ID}" \
  --region="${REGION}" \
  --filter="metadata.verb:upgrade"
```

```bash
gcloud alloydb operations describe "<OPERATION_ID>" \
  --region="${REGION}" \
  --project="${PROJECT_ID}"
```

---

## 6. Phase 4 — Validate Clone

Do not perform generic testing. Execute specific validation:

### 6.1 Version Check
```sql
SELECT version();
SHOW server_version;
```

### 6.2 Specific Application Validation Steps
* **Application connects:** Connection pools successfully authenticate and connect.
* **Existing databases accessible:** All schemas, tables, and views can be queried.
* **Extensions load:** All installed extensions load properly in PostgreSQL 18.
* **Critical queries work:** Core application queries execute with valid query execution plans.
* **Read/write operations work:** Verified INSERT, UPDATE, and DELETE operations complete successfully.
* **Application logs checked:** Zero fatal or syntax error logs related to deprecated PostgreSQL features or driver incompatibilities.
* **Monitoring checked:** CPU, memory, and disk metrics stabilize at normal operating baseline.

> **"Application is successfully connected to the upgraded PostgreSQL 18 clone, required data is accessible, and critical application workflows are working as expected."Then Only after this validation is successful should production upgrade proceed.**

---

## 7. Phase 5 — Production Upgrade Preparation

Create an on-demand pre-upgrade backup of the production cluster:

```bash
gcloud alloydb backups create "${BACKUP_ID}" \
  --cluster="${PROD_CLUSTER_ID}" \
  --region="${REGION}" \
  --project="${PROJECT_ID}" \
  --async
```

Monitor the backup operation until completion:
```bash
gcloud alloydb operations describe "<OPERATION_ID>" \
  --region="${REGION}" \
  --project="${PROJECT_ID}"
```

> **CRITICAL RULE:**  
> **Do not start the production upgrade until the backup operation has completed successfully.**

---

## 8. Phase 6 — Actual Production In-Place Upgrade

Execute the in-place upgrade on the production cluster:

```bash
gcloud alloydb clusters upgrade "${PROD_CLUSTER_ID}" \
  --region="${REGION}" \
  --version=POSTGRES_18 \
  --async
```

### What Happens Internally to the Existing Production Cluster:
```
Existing Production Cluster
        |
        | PostgreSQL 14
        |
        | Upgrade initiated
        ↓
Pre-upgrade validation
        ↓
Upgrade preparation
        ↓
Primary instance upgrade
        ↓
PostgreSQL 18 starts
        ↓
Read pools upgraded if applicable
        ↓
Cleanup
        ↓
Production Cluster
        |
        └── PostgreSQL 18
```

*The existing production cluster itself is upgraded in-place rather than creating another production cluster and migrating data.*

---

## 9. Phase 7 — Monitor Production Upgrade

### Monitoring Commands:
```bash
gcloud alloydb operations list \
  --cluster="${PROD_CLUSTER_ID}" \
  --region="${REGION}" \
  --filter="metadata.verb:upgrade"
```

```bash
gcloud alloydb operations describe "<OPERATION_ID>" \
  --region="${REGION}" \
  --project="${PROJECT_ID}"
```

### Upgrade Status Meanings & Action Plan:

* **SUCCESS:**
  * **Meaning:** The upgrade completed cleanly across the primary instance and read pools.
  * **Action:** Proceed immediately to Phase 8 Post-Upgrade Validation.

* **FAILED:**
  * **Meaning:** Pre-upgrade validation checks or core engine upgrade failed.
  * **Action:** AlloyDB rolls back to PostgreSQL 14. Inspect error details in the operation description and Cloud Logging. Keep production on PG14 while resolving blockers.

* **PARTIAL_SUCCESS:**
  * **Meaning:** Primary instance upgraded successfully to PostgreSQL 18, but one or more read pools encountered errors during upgrade.
  * **Action:** Primary database is operational on PG18. Inspect instance health, remove failed read pool instances, and recreate them under the upgraded cluster.

---

## 10. Phase 8 — Post-Upgrade Validation

### 10.1 Version Verification
Immediately after completion, run:
```sql
SELECT version();
SHOW server_version;
```
* **Expected Result:** `PostgreSQL 18`

### 10.2 Instance State Verification
```bash
gcloud alloydb instances list \
  --cluster="${PROD_CLUSTER_ID}" \
  --region="${REGION}" \
  --project="${PROJECT_ID}"
```

### 10.3 Application Validation
* Confirm application connects to the primary endpoint.
* Verify all user databases and tables are accessible.
* Run end-to-end read and write transactional tests.
* Inspect application logs and performance monitoring.

---

## 11. Recovery / Rollback

### Technical Reality
**PostgreSQL 18 cannot simply be downgraded in-place back to PostgreSQL 14.**

If recovery is required, follow this procedure:

```
Pre-upgrade backup
       ↓
Restore into NEW AlloyDB cluster
       ↓
Validate recovered PostgreSQL environment
       ↓
Redirect application
```

### Recovery Command:
```bash
gcloud alloydb clusters restore "<RECOVERY_CLUSTER_ID>" \
  --backup="<PRE_UPGRADE_BACKUP_ID>" \
  --region="${REGION}" \
  --project="${PROJECT_ID}"
```

After restoring the cluster, provision the primary instance, validate connectivity, and redirect application traffic to the newly restored cluster.

---

## 12. Implementation Completion

### End-to-End Workflow Summary:
```
PG14 Production
      ↓
Pre-checks
      ↓
PITR Clone
      ↓
Clone PG14 → PG18
      ↓
Application Validation
      ↓
Production Backup
      ↓
Production PG14 → PG18
      ↓
Post-upgrade Validation
      ↓
Production PG18
```
