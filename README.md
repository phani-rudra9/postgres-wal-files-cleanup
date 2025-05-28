# PostgreSQL WAL Files Cleanup - Complete Guide

## Overview

This document provides a comprehensive guide for automatically cleaning up unused PostgreSQL Write-Ahead Log (WAL) files in a Kubernetes environment. The solution uses a Kubernetes CronJob that runs daily to safely remove WAL files older than 10 days while preserving files that are still needed by PostgreSQL.

## What are WAL Files?

Write-Ahead Log (WAL) files are PostgreSQL's transaction log files that ensure data integrity and enable point-in-time recovery. Over time, these files accumulate and can consume significant disk space if not properly managed.

**Key characteristics:**
- WAL files are created continuously as PostgreSQL processes transactions
- Each WAL file is typically 16MB in size
- Files are named with hexadecimal sequences (e.g., `000000010000000000000001`)
- PostgreSQL needs recent WAL files for recovery and replication

**WAL Archiving Process:**
When PostgreSQL has WAL archiving enabled, it follows this lifecycle:
1. PostgreSQL writes transactions to active WAL file
2. When WAL file is complete, PostgreSQL creates a `.ready` file in `archive_status/`
3. Archive process copies the WAL file to backup location
4. After successful archiving, `.ready` file becomes `.done` file
5. WAL files with `.done` status are safe to delete (they're backed up)

## Why This Solution?

**Problems solved:**
- Prevents disk space exhaustion from accumulating WAL files
- Maintains data integrity by only deleting safe-to-remove files
- Provides automated, hands-off maintenance
- Follows Kubernetes best practices for scheduled tasks

**Safety measures:**
- Connects to PostgreSQL to determine currently active WAL files
- Only deletes files older than specified retention period (10 days)
- Forces PostgreSQL checkpoint before cleanup
- Preserves all files needed for recovery

## Prerequisites

Before implementing this solution, ensure you have:

1. **Kubernetes cluster** with batch/v1 API support
2. **PostgreSQL running in Kubernetes** with persistent storage
3. **Access to PostgreSQL credentials** stored in Kubernetes secrets
4. **Appropriate RBAC permissions** to create CronJobs
5. **Persistent Volume Claim (PVC)** for PostgreSQL data

## Implementation

### Step 1: Prepare the Environment

First, identify your PostgreSQL resources:

```bash
kubectl get pods -n your-namespace | grep postgres

kubectl get pvc -n your-namespace | grep postgres

kubectl get svc -n your-namespace | grep postgres

kubectl get secrets -n your-namespace | grep postgres
```

### Step 2: Create the CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-wal-cleanup
  namespace: your-namespace
spec:
  schedule: "0 0 * * *"
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: wal-cleanup
            image: postgres:15
            env:
            - name: PGHOST
              value: "postgresql-medcomp-pactbroker-dev"
            - name: PGUSER
              value: "postgres"
            - name: PGDATABASE
              value: "postgres"
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            command:
            - /bin/bash
            - -c
            - |
              echo "Starting WAL cleanup at $(date)"
              
              psql -c "SELECT pg_switch_wal();"
              psql -c "CHECKPOINT;"
              
              cd /srv/postgresql/volume/wal
              
              CURRENT_WAL=$(psql -t -c "SELECT pg_walfile_name(pg_current_wal_lsn());" | tr -d ' ')
              echo "Current WAL: $CURRENT_WAL"
              
              DELETED_COUNT=0
              
              for wal_file in *.wal; do
                if [ -f "$wal_file" ] && [ "$wal_file" != "$CURRENT_WAL" ]; then
                  
                  if [ -f "archive_status/${wal_file}.done" ]; then
                    if [ $(find . -name "$wal_file" -mtime +10 | wc -l) -gt 0 ]; then
                      echo "Deleting archived WAL: $wal_file"
                      rm -f "$wal_file"
                      rm -f "archive_status/${wal_file}.done"
                      DELETED_COUNT=$((DELETED_COUNT + 1))
                    fi
                  else
                    echo "Skipping $wal_file (not archived yet or still needed)"
                  fi
                fi
              done
              
              find . -name "*.backup" -mtime +10 -delete
              find archive_status/ -name "*.done" -mtime +10 -delete
              
              echo "WAL cleanup completed. Deleted $DELETED_COUNT files at $(date)"
            
            volumeMounts:
            - name: postgres-data
              mountPath: /srv/postgresql/volume
          
          volumes:
          - name: postgres-data
            persistentVolumeClaim:
              claimName: postgresql-medcomp-pactbroker-dev-pvc
          
          restartPolicy: OnFailure
```

### Step 3: Deploy the CronJob

```bash
kubectl apply -f postgres-wal-cleanup.yaml

kubectl get cronjobs -n your-namespace

kubectl describe cronjob postgres-wal-cleanup -n your-namespace
```

## Detailed Script Explanation

### CronJob Configuration Section

```yaml
apiVersion: batch/v1
kind: CronJob
```
**Purpose:** Defines this as a Kubernetes CronJob resource that runs scheduled tasks.

```yaml
schedule: "0 0 * * *"
```
**Purpose:** Sets the execution schedule using cron format.
**Breakdown:** 
- `0` = minute (0th minute)
- `0` = hour (0th hour, midnight)
- `*` = any day of month
- `*` = any month
- `*` = any day of week
**Result:** Runs daily at 12:00 AM

```yaml
successfulJobsHistoryLimit: 3
failedJobsHistoryLimit: 3
```
**Purpose:** Kubernetes keeps history of job executions for monitoring and debugging.
**Benefit:** Prevents unlimited accumulation of job history while maintaining visibility into recent executions.

### Container Configuration Section

```yaml
image: postgres:15
```
**Purpose:** Uses the official PostgreSQL Docker image which includes all PostgreSQL client tools.
**Why needed:** Provides `psql` command and PostgreSQL functions needed for safe WAL management.

```yaml
env:
- name: PGHOST
  value: "postgresql-medcomp-pactbroker-dev"
```
**Purpose:** Sets the PostgreSQL server hostname for connection.
**Important:** This should match your PostgreSQL service name in Kubernetes.

```yaml
- name: PGUSER
  value: "postgres"
- name: PGDATABASE
  value: "postgres"
```
**Purpose:** Defines the database user and database name for PostgreSQL connection.
**Security:** Uses standard postgres superuser for administrative tasks.

```yaml
- name: PGPASSWORD
  valueFrom:
    secretKeyRef:
      name: postgres-secret
      key: password
```
**Purpose:** Securely retrieves PostgreSQL password from Kubernetes secret.
**Security benefit:** Avoids hardcoding passwords in YAML files.

### Volume Mounting Section

```yaml
volumeMounts:
- name: postgres-data
  mountPath: /srv/postgresql/volume

volumes:
- name: postgres-data
  persistentVolumeClaim:
    claimName: postgresql-medcomp-pactbroker-dev-pvc
```
**Purpose:** Mounts the PostgreSQL data volume into the cleanup container.
**Critical:** This gives the cleanup script access to the WAL files directory.

### Script Logic Breakdown

```bash
echo "Starting WAL cleanup at $(date)"
```
**Purpose:** Logs the start time for monitoring and debugging.
**Output example:** `Starting WAL cleanup at Wed May 28 00:00:01 UTC 2025`

```bash
psql -c "SELECT pg_switch_wal();"
```
**Purpose:** Forces PostgreSQL to switch to a new WAL file.
**Why important:** Makes the current WAL file available for potential cleanup and ensures a clean state.
**PostgreSQL function:** `pg_switch_wal()` is a built-in PostgreSQL administrative function.

```bash
psql -c "CHECKPOINT;"
```
**Purpose:** Forces PostgreSQL to write all dirty buffers to disk and update control files.
**Why critical:** Ensures that WAL files are no longer needed for crash recovery before deletion.
**Effect:** Makes old WAL files safe to remove.

```bash
cd /srv/postgresql/volume/wal
```
**Purpose:** Changes to the directory containing WAL files.
**Path explanation:** This matches your PostgreSQL WAL directory structure.

```bash
CURRENT_WAL=$(psql -t -c "SELECT pg_walfile_name(pg_current_wal_lsn());" | tr -d ' ')
```
**Purpose:** Identifies the currently active WAL file that PostgreSQL is writing to.
**Breakdown:**
- `psql -t` = Tuples-only mode (no headers or row count)
- `pg_current_wal_lsn()` = Gets current Write-Ahead Log Location Sequence Number
- `pg_walfile_name()` = Converts LSN to actual WAL filename
- `tr -d ' '` = Removes any whitespace from the result
**Safety:** This file must NEVER be deleted as PostgreSQL is actively using it.

```bash
DELETED_COUNT=0
```
**Purpose:** Initialize a counter to track how many files are deleted for reporting.

```bash
for wal_file in *.wal; do
```
**Purpose:** Iterates through all WAL files in the directory.
**Pattern matching:** `*.wal` matches all files ending with `.wal` extension.

```bash
if [ -f "$wal_file" ] && [ "$wal_file" != "$CURRENT_WAL" ]; then
```
**Purpose:** Double safety check before considering a file for deletion.
**Conditions:**
- `[ -f "$wal_file" ]` = Verifies it's a regular file (not a directory or symlink)
- `[ "$wal_file" != "$CURRENT_WAL" ]` = Ensures it's not the currently active WAL file

```bash
if [ -f "archive_status/${wal_file}.done" ]; then
```
**Purpose:** NEW SAFETY CHECK - Only consider files that have been successfully archived.
**Why critical:** Files with `.done` status have been backed up and are safe to remove.
**Archive status:** PostgreSQL creates `.done` files when WAL files are successfully archived.

```bash
if [ $(find . -name "$wal_file" -mtime +10 | wc -l) -gt 0 ]; then
```
**Purpose:** Checks if the WAL file is older than 10 days.
**Breakdown:**
- `find . -name "$wal_file" -mtime +10` = Finds files modified more than 10 days ago
- `wc -l` = Counts the number of lines (files found)
- `-gt 0` = Greater than 0 (meaning the file is indeed older than 10 days)

```bash
echo "Deleting archived WAL: $wal_file"
rm -f "$wal_file"
rm -f "archive_status/${wal_file}.done"
DELETED_COUNT=$((DELETED_COUNT + 1))
```
**Purpose:** Logs the deletion action and removes both the WAL file and its status file.
**Safety:** Only executes after confirming the file is archived, old, and not currently active.
**Cleanup:** Removes both the WAL file and corresponding `.done` status file.
**Counting:** Increments the deletion counter for reporting.

```bash
else
  echo "Skipping $wal_file (not archived yet or still needed)"
```
**Purpose:** Logs when files are skipped and explains why.
**Safety:** Files without `.done` status are either still being written to or not yet archived.

```bash
find . -name "*.backup" -mtime +10 -delete
find archive_status/ -name "*.done" -mtime +10 -delete
```
**Purpose:** 
- First line: Removes PostgreSQL backup label files older than 10 days
- Second line: Cleans up orphaned `.done` status files that might remain after manual deletions
**Why safe:** Backup files and old status files are metadata that can be safely removed after the retention period.

```bash
echo "WAL cleanup completed. Deleted $DELETED_COUNT files at $(date)"
```
**Purpose:** Logs completion time and reports how many files were actually deleted for monitoring purposes.

## Testing and Validation

### Manual Testing

Before relying on the scheduled execution, test the CronJob manually:

```bash
kubectl create job --from=cronjob/postgres-wal-cleanup manual-wal-cleanup -n your-namespace

kubectl get jobs -n your-namespace -w

kubectl logs job/manual-wal-cleanup -n your-namespace

kubectl delete job manual-wal-cleanup -n your-namespace
```

### Monitoring CronJob Status

```bash
kubectl get cronjobs postgres-wal-cleanup -n your-namespace

kubectl get jobs -n your-namespace | grep postgres-wal-cleanup

kubectl logs -l job-name=postgres-wal-cleanup-$(date +%s) -n your-namespace
```

## Customization Options

### Adjusting Retention Period

To change the retention period from 10 days to a different value:

```bash
# For 7 days retention
-mtime +7

# For 30 days retention  
-mtime +30
```

### Changing Schedule

Common schedule patterns:

```yaml
# Every 6 hours
schedule: "0 */6 * * *"

# Weekly on Sunday at 2 AM
schedule: "0 2 * * 0"

# Monthly on the 1st at midnight
schedule: "0 0 1 * *"
```

### Adding Notifications

To add Slack or email notifications on job completion:

```yaml
- name: notify
  image: curlimages/curl
  command:
  - /bin/sh
  - -c
  - |
    curl -X POST -H 'Content-type: application/json' \
    --data '{"text":"PostgreSQL WAL cleanup completed"}' \
    YOUR_SLACK_WEBHOOK_URL
```
