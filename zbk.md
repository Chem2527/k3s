# Zenus QA Environment — Database & Infrastructure Migration Checklist (`todo.md`)

This document provides the step-by-step execution guide for implementing PgBouncer connection pooling, Azure PostgreSQL server parameters, Azure DevOps variable group updates, and Azure Container Apps (ACA) compute autoscaling rules **specifically for the QA environment** (`zb-psql-pp-qa-eastus-001.postgres.database.azure.com`).

---

## 1. Microservice Repositories & Variable Group Mapping (QA)

The following 8 microservice repositories and their associated Azure DevOps Variable Groups interact directly with the QA PostgreSQL database:

| # | Microservice Repository Name | Azure DevOps Variable Group (QA) | Linked Azure Container App (QA) |
|---|---|---|---|
| 1 | `fintech_user_management` | `ZB-FintechUserManagment-QA` | `zb-qa-pp-user-management-001` |
| 2 | `fintech_super_admin` | `ZB-FintechSupeAdmin-QA` | `zb-qa-pp-super-admin-001` |
| 3 | `fintech_business_management` | `ZB-FintechBusinessManagment-QA` | `zb-qa-pp-business-management-001` |
| 4 | `fintech_business_settings` | `ZB-FintechBusinessSettings-QA` | `zb-qa-pp-business-settings-001` |
| 5 | `fintech_notifications_management` | `ZB-FintechNotificationsManagement-QA` | `zb-qa-pp-notifications-management-001` |
| 6 | `fintech_documents_management` | `ZB-FintechDocumentsManagement-QA` | `zb-qa-pp-documents-management-001` |
| 7 | `fintech_management_migrations` | `ZB-FintechManagementMigrations-QA` | Executed during CD deployment pipeline |
| 8 | `fintech_processing_scripts` | `ZB-FintechProcessingScripts-QA` | Executed via Azure Cron / Batch jobs |

---

## 2. Current vs. Target Configurations

### A. Azure DevOps Variable Groups (QA)

#### Current Settings (Audited via Azure DevOps API):
* `DB_HOST`: `zb-psql-pp-qa-eastus-001.postgres.database.azure.com`
* `DB_PORT`: `5432` *(Direct unpooled PostgreSQL port)*
* `DB_MAX_CLIENTS`: `20` (and `40` for documents-mgnt)
* `DB_USER`: `fintech_user`
* `DB_SCHEMA`: `fintech`

#### Target Settings (What needs to be changed):
| Variable Group | Variable | Current Value | **Target Value (QA)** | Why Change? |
|---|---|---|---|---|
| `ZB-FintechUserManagment-QA` | `DB_PORT` | `5432` | **`6432`** | Routes connections through PgBouncer pooler instead of direct DB. |
| `ZB-FintechUserManagment-QA` | `DB_MAX_CLIENTS` | `20` | **`3`** | Caps client sockets per container instance to prevent pool hogging. |
| `ZB-FintechSupeAdmin-QA` | `DB_PORT` | `5432` | **`6432`** | Routes connections through PgBouncer pooler. |
| `ZB-FintechSupeAdmin-QA` | `DB_MAX_CLIENTS` | `20` | **`3`** | Caps client sockets per container instance. |
| `ZB-FintechBusinessManagment-QA` | `DB_PORT` | `5432` | **`6432`** | Routes connections through PgBouncer pooler. |
| `ZB-FintechBusinessManagment-QA` | `DB_MAX_CLIENTS` | `20` | **`3`** | Caps client sockets per container instance. |
| `ZB-FintechBusinessSettings-QA` | `DB_PORT` | `5432` | **`6432`** | Routes connections through PgBouncer pooler. |
| `ZB-FintechBusinessSettings-QA` | `DB_MAX_CLIENTS` | `20` | **`3`** | Caps client sockets per container instance. |
| `ZB-FintechNotificationsManagement-QA` | `DB_PORT` | `5432` | **`6432`** | Routes connections through PgBouncer pooler. |
| `ZB-FintechNotificationsManagement-QA` | `DB_MAX_CLIENTS` | `20` | **`3`** | Caps client sockets per container instance. |
| `ZB-FintechDocumentsManagement-QA` | `DB_PORT` | `5432` | **`6432`** | Routes connections through PgBouncer pooler. |
| `ZB-FintechDocumentsManagement-QA` | `DB_MAX_CLIENTS` | `40` | **`3`** | Caps client sockets per container instance. |

> [!NOTE]
> **Migrations & Scripts Note**: 
> For `ZB-FintechManagementMigrations-QA` and `ZB-FintechProcessingScripts-QA`, keep `DB_PORT = 5432` (Direct port), because DDL migrations (schema changes/table locks) require direct connections and cannot run in transaction-pooled PgBouncer mode.

---

### B. Azure Container Apps (ACA) Configuration (QA)

#### Current State (Audited from Azure Telemetry):
* `minReplicas`: `1`
* `maxReplicas`: `10`
* `Scale Rules`: **HTTP concurrency rule only** (`concurrentRequests = 10`).
* `CPU Scale Rule`: **NONE** (0 CPU rules configured).
* `Memory Scale Rule`: **NONE** (0 Memory rules configured).

#### Problem with Current ACA State:
In QA telemetry, microservices `zb-qa-pp-user-management-001` and `zb-qa-pp-super-admin-001` spiked to **102%–410% CPU**, but because socket concurrency was `<10`, ACA scale engine evaluated `scaleTarget = 0` and **refused to scale beyond 1 replica**.

#### Target ACA Configuration (What to Add in QA):
1. **Retain HTTP Rule**: `concurrentRequests = 10`.
2. **Add CPU Scale Rule**: Metric = `CPU utilization`, Target Threshold = **`70%`**.
3. **Add Memory Scale Rule**: Metric = `Memory utilization`, Target Threshold = **`75%`**.

---

### C. Azure PostgreSQL Server Parameters (QA Server: `zb-psql-pp-qa-eastus-001`)

#### Current vs Target State:

| Parameter Name | Current Value | **Target Value (QA)** | Rationale & Purpose |
|---|---|---|---|
| `pgbouncer.enabled` | `OFF` | **`true` (Port `6432`)** | Enables PgBouncer built-in pooler on port 6432. |
| `pgbouncer.pool_mode` | `transaction` | **`transaction`** | Reuses DB connections immediately when a transaction completes (5ms–20ms). |
| `pgbouncer.min_pool_size` | `0` | **`2`** | Pre-warms 2 standby connections 24/7 so QA dev testing has zero connection delay. |
| `pgbouncer.default_pool_size` | `20` | **`15`** | Max 15 warm backend connections to PostgreSQL. Highly optimal for QA load. |
| `pgbouncer.max_client_conn` | `5000` | **`5000`** | Allows up to 5,000 incoming client sockets to connect to PgBouncer. |
| `idle_in_transaction_session_timeout` | `0` (Disabled) | **`30000` (30 sec)** | Kills abandoned transactions idle >30s. Writes error code `57P01` to Azure logs. |
| `lock_timeout` | `0` (Disabled) | **`10000` (10 sec)** | Kills blocked row lock waits >10s. Writes error code `55P03` to Azure logs. |
| `statement_timeout` | `0` (Disabled) | **`60000` (60 sec)** | Kills runaway unindexed queries >60s. Writes error code `57014` to Azure logs. |

---

## 3. Override Hierarchy: How Values Get Picked

1. **Azure DevOps Variable Groups (`ZB-Fintech*-QA`)**:
   * We set `DB_PORT = 6432` and `DB_MAX_CLIENTS = 3`.
2. **Container Environment Variables (ACA Runtime)**:
   * When CD pipeline deploys, Azure DevOps injects these values into `process.env.DB_PORT` and `process.env.DB_MAX_CLIENTS` inside the ACA container.
   * **Container environment variables OVERRIDE any code default fallbacks**.
3. **Microservice Code (Node.js/Knex)**:
   * The app opens 3 client sockets to PgBouncer port 6432.
4. **PgBouncer (Port 6432)**:
   * PgBouncer queues requests and multiplexes them down to `15` warm backend connections to PostgreSQL port 5432.

---

## 4. Step-by-Step QA Execution Plan

### Step 1: Update Azure PostgreSQL Server Parameters (QA)
1. Open **Azure Portal** -> **Azure Database for PostgreSQL Flexible Server** -> `zb-psql-pp-qa-eastus-001`.
2. Navigate to **Server Parameters**.
3. Set `pgbouncer.enabled` = `true`.
4. Set `pgbouncer.min_pool_size` = `2`.
5. Set `pgbouncer.default_pool_size` = `15`.
6. Set `idle_in_transaction_session_timeout` = `30000`.
7. Set `lock_timeout` = `10000`.
8. Set `statement_timeout` = `60000`.
9. Click **Save**.

### Step 2: Update Azure DevOps Variable Groups (QA)
1. Open **Azure DevOps** -> Project `ZB-CS - PP Digital Portal Solution` -> **Pipelines** -> **Library**.
2. Update the 6 QA Variable Groups (`ZB-FintechUserManagment-QA`, `ZB-FintechSupeAdmin-QA`, `ZB-FintechBusinessManagment-QA`, `ZB-FintechBusinessSettings-QA`, `ZB-FintechNotificationsManagement-QA`, `ZB-FintechDocumentsManagement-QA`):
   * Set `DB_PORT` = `6432`.
   * Set `DB_MAX_CLIENTS` = `3`.
3. Save each variable group.

### Step 3: Add ACA CPU & Memory Scale Rules (QA)
1. In Azure Portal, navigate to Container Apps: `zb-qa-pp-user-management-001`, `zb-qa-pp-super-admin-001`, `zb-qa-pp-business-management-001`, `zb-qa-pp-business-settings-001`, `zb-qa-pp-notifications-management-001`, `zb-qa-pp-documents-management-001`.
2. Under **Scale**, add two custom scale rules:
   * **CPU Rule**: Name = `cpu-scale-rule`, Type = `Custom`, Metric = `cpu`, Utilization = `70%`.
   * **Memory Rule**: Name = `memory-scale-rule`, Type = `Custom`, Metric = `memory`, Utilization = `75%`.
3. Click **Save / Create Revision**.

### Step 4: Trigger QA CD Deployments
1. In Azure DevOps, run the CD pipelines for the 6 QA microservices to redeploy containers with updated environment variables (`DB_PORT=6432`).

### Step 5: Execute Read-Only PostgreSQL Audit
Connect to QA PostgreSQL server via pgAdmin/psql and run the following **READ-ONLY** SQL script:

```sql
-- Verify PgBouncer & Defensive Parameter Settings
SHOW pgbouncer.enabled;
SHOW pgbouncer.min_pool_size;
SHOW pgbouncer.default_pool_size;
SHOW idle_in_transaction_session_timeout;
SHOW lock_timeout;
SHOW statement_timeout;

-- Verify Active Connections are routed via PgBouncer Port 6432
SELECT 
    state, 
    usename, 
    client_addr, 
    count(*) AS open_connections
FROM pg_stat_activity
GROUP BY state, usename, client_addr
ORDER BY open_connections DESC;
```
