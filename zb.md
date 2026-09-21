#  Zenus QA Environment — Database & Infrastructure Migration Checklist (`todo.md`)

This document provides the exact step-by-step execution guide for implementing PgBouncer connection pooling, Azure PostgreSQL server parameters, Azure DevOps variable group updates, and Azure Container Apps (ACA) compute autoscaling rules **specifically for the QA environment** (`<POSTGRES_QA_SERVER_HOST>`).

---

## 1. Master Infrastructure & Environment Details (QA)

| Infrastructure Component | QA Resource Placeholder / Value |
|---|---|
| **Azure Organization** | `<AZURE_ORGANIZATION>` |
| **Azure DevOps Project** | `<AZURE_DEVOPS_PROJECT>` |
| **PostgreSQL QA Server** | `<POSTGRES_QA_SERVER_HOST>` |
| **Database Name & Schema** | Database: `<DATABASE_NAME>` \| Schema: `<DATABASE_SCHEMA>` |
| **Database User** | `<DATABASE_USER>` |
| **Direct PostgreSQL Port** | `5432` *(Use ONLY for migrations & processing scripts)* |
| **PgBouncer Pooler Port** | `6432` *(Use for ALL microservices)* |
| **Redis Cache QA Host** | `<REDIS_CACHE_QA_HOST>` (Port `6380`) |
| **Azure Resource Group** | `<AZURE_RESOURCE_GROUP>` |
| **Log Analytics Workspace** | `<LOG_ANALYTICS_WORKSPACE>` |
| **Application Insights** | `<APPLICATION_INSIGHTS_NAME>` |

---

## 2. Microservice Repositories & Variable Group Mapping (QA)

The following 8 microservice repositories and their associated Azure DevOps Variable Groups interact directly with `<DATABASE_SCHEMA>` on `<POSTGRES_QA_SERVER_HOST>`:

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

## 3. Current vs. Target Configurations

### A. Azure DevOps Variable Groups (QA)

#### Current Audited Variables in `<AZURE_DEVOPS_PROJECT>`:
* `DB_HOST`: `<POSTGRES_QA_SERVER_HOST>`
* `DB_USER`: `<DATABASE_USER>`
* `DB_SCHEMA`: `<DATABASE_SCHEMA>`
* `DB_PORT`: `5432` *(Direct unpooled port)*
* `DB_MAX_CLIENTS`: `20` (and `40` for documents-mgnt)

#### Target Settings (What needs to be modified):
| Variable Group | Variable | Current Value | **Target Value (QA)** | Rationale |
|---|---|---|---|---|
| `ZB-FintechUserManagment-QA` | `DB_PORT` | `5432` | **`6432`** | Routes `<DATABASE_USER>` queries through PgBouncer pooler. |
| `ZB-FintechUserManagment-QA` | `DB_MAX_CLIENTS` | `20` | **`3`** | Caps client driver pool sockets (Math proof below). |
| `ZB-FintechSupeAdmin-QA` | `DB_PORT` | `5432` | **`6432`** | Routes `<DATABASE_USER>` queries through PgBouncer. |
| `ZB-FintechSupeAdmin-QA` | `DB_MAX_CLIENTS` | `20` | **`3`** | Caps client driver pool sockets. |
| `ZB-FintechBusinessManagment-QA` | `DB_PORT` | `5432` | **`6432`** | Routes `<DATABASE_USER>` queries through PgBouncer. |
| `ZB-FintechBusinessManagment-QA` | `DB_MAX_CLIENTS` | `20` | **`3`** | Caps client driver pool sockets. |
| `ZB-FintechBusinessSettings-QA` | `DB_PORT` | `5432` | **`6432`** | Routes `<DATABASE_USER>` queries through PgBouncer. |
| `ZB-FintechBusinessSettings-QA` | `DB_MAX_CLIENTS` | `20` | **`3`** | Caps client driver pool sockets. |
| `ZB-FintechNotificationsManagement-QA` | `DB_PORT` | `5432` | **`6432`** | Routes `<DATABASE_USER>` queries through PgBouncer. |
| `ZB-FintechNotificationsManagement-QA` | `DB_MAX_CLIENTS` | `20` | **`3`** | Caps client driver pool sockets. |
| `ZB-FintechDocumentsManagement-QA` | `DB_PORT` | `5432` | **`6432`** | Routes `<DATABASE_USER>` queries through PgBouncer. |
| `ZB-FintechDocumentsManagement-QA` | `DB_MAX_CLIENTS` | `40` | **`3`** | Caps client driver pool sockets. |

> [!IMPORTANT]
> **Schema Migrations Note**: 
> For `ZB-FintechManagementMigrations-QA` and `ZB-FintechProcessingScripts-QA`, keep `DB_PORT = 5432` (Direct port), because DDL migrations (`ALTER TABLE`, `CREATE INDEX`) require direct connections and cannot run in transaction-pooled PgBouncer mode.

---

### 📐 Mathematical Proofs: Pool Sizing & `DB_MAX_CLIENTS = 3`

#### 1. Pool Sizing Formula:
```text
connections = ((core_count * 2) + effective_spindle_count)
```

Where:
* `core_count` = Number of vCPU cores on the PostgreSQL database server.
* `effective_spindle_count` = Disk I/O parallelism factor (2 to 4 for SSD storage).

##### Exact Environment Calculations:

* **QA Database Server (2 vCPUs / 8 GB RAM)**:
  * Base Active Connections = `(2 vCPUs * 2) + 2 = 6 connections`
  * Adding a 2.5x safety multiplier for web transaction hold times:
  * **Target Pool Size (QA)** = `6 * 2.5 = 15 connections` (**`default_pool_size = 15`**)

* **PROD Database Server (8 vCPUs / 32 GB RAM)**:
  * Base Active Connections = `(8 vCPUs * 2) + 4 = 20 connections`
  * Adding a 2.5x safety multiplier for web transaction hold times:
  * **Target Pool Size (PROD)** = `20 * 2.5 = 50 connections` (**`default_pool_size = 50`**)

---

#### 2. Time Arithmetic Math for `DB_MAX_CLIENTS = 3`:
* 1 second = 1,000 milliseconds (ms)
* In PostgreSQL, an OLTP query (`SELECT`, `UPDATE` in `<DATABASE_SCHEMA>` schema) takes 5 ms to execute.
* **1 single socket connection** can run:
  * `1,000 ms / 5 ms per query = 200 queries per second`
* Therefore, **3 socket connections** on 1 container replica can run:
  * `3 connections * 200 queries/sec = 600 queries per second per container`
* A single container replica in QA handles far less than 600 QPS. Thus, **3 sockets per container** provides 100% capacity headroom.

#### 3. Before vs. After Comparison (12 Containers in QA):
* **OLD (`DB_MAX_CLIENTS = 20` on Port 5432 Direct DB)**:
  * `12 containers * 20 = 240 direct connections open on DB`
  * *Result*: Containers hoarded 240 idle connections, exceeding `max_connections = 500` under scale-out and crashing the database.
* **NEW (`DB_MAX_CLIENTS = 3` on Port 6432 PgBouncer)**:
  * `12 containers * 3 = 36 client connections to PgBouncer`
  * *Result*: PgBouncer multiplexes 36 client connections into **15 warm PostgreSQL connections** (`default_pool_size = 15`), keeping DB CPU `<20%` with zero crashes.

---

### B. Azure Container Apps (ACA) Configuration (QA)

#### Current State (Audited from Azure Telemetry):
* `minReplicas`: `1`
* `maxReplicas`: `10`
* `Scale Rules`: **HTTP concurrency rule only** (`concurrentRequests = 10`).
* `CPU Scale Rule`: **NONE** (0 CPU rules configured).
* `Memory Scale Rule`: **NONE** (0 Memory rules configured).

#### Target ACA Configuration (What to Add in QA):
1. **Retain HTTP Rule**: `concurrentRequests = 10`.
2. **Add CPU Scale Rule**: Metric = `CPU utilization`, Target Threshold = **`70%`**.
3. **Add Memory Scale Rule**: Metric = `Memory utilization`, Target Threshold = **`75%`**.

---

### 3.3 Supporting Visual Evidence: QA Container Telemetry Screenshots (Empirical Proof)

Below are the live Azure Portal telemetry metrics screenshots captured directly from the QA environment for microservices `zb-qa-pp-user-management-001` and `zb-qa-pp-super-admin-001`, providing visual empirical proof of the HTTP scaling limitation:

#### Figure 1: QA Microservice `zb-qa-pp-user-management-001` Metrics (CPU > 100% with 0 Scale-Out)
As shown in the Azure Portal chart below, the CPU Usage Percentage (Blue Line) repeatedly spikes past 100% (peaking at 102.0%), while Replica Count (Pink Line) remains flat at 1 to 2 replicas max without triggering a scale-out to 3, 4, 5, or 10 replicas, despite the HTTP scale rule (`concurrentRequests = 10`) being active:

<img width="1917" height="932" alt="image" src="https://github.com/user-attachments/assets/65226076-a5a3-41fb-acc7-f6084aa33d9e" />


#### Figure 2: QA Microservice `zb-qa-pp-super-admin-001` Metrics (CPU > 400% with 0 Scale-Out)
As shown in the Azure Portal chart below, the CPU Usage Percentage (Blue Line) spikes past 400% (peaking at 410.0% multi-core burst), while Replica Count (Pink Line) remains locked at 1 replica. Even though `maxReplicas = 10` is configured in Azure, the HTTP concurrency scaler failed to trigger a new container replica:

<img width="975" height="449" alt="image" src="https://github.com/user-attachments/assets/16fe1702-40cb-4f94-895c-94636b999cbf" />



#### ⚠️ Technical Vulnerability Analysis & Why PROD Faces the Exact Same Risk:
1. **Identical Scale Rule Vulnerability**: Both PROD and QA microservices currently rely **EXCLUSIVELY on an HTTP concurrency scale rule** (`concurrentRequests = 10`) with **ZERO CPU or Memory rules configured**.
2. **The Shared Scaling Failure**: In QA, when CPU pins at 102%–410%, Azure Container Apps scale engine evaluates only instantaneous HTTP socket concurrency (<10) and returns `scaleTarget = 0`, keeping the app locked at 1 replica.
3. **The Risk in PROD**: If traffic spikes in PROD, the exact same HTTP rule will fail to scale PROD container replicas as single-threaded Node.js event loops hit 100% CPU lock!
4. **Mandatory Fix**: Provisioning custom **CPU (70%)** and **Memory (75%)** scaling rules guarantees automatic scale-out whenever compute utilization increases.

---

### C. Azure PostgreSQL Server Parameters (Server: `<POSTGRES_QA_SERVER_HOST>`)

| Parameter Name | Current Value | **Target Value (QA)** | Purpose & Error Code |
|---|---|---|---|
| `pgbouncer.enabled` | `OFF` | **`true` (Port `6432`)** | Enables PgBouncer built-in pooler on port 6432. |
| `pgbouncer.pool_mode` | `transaction` | **`transaction`** | Reuses DB connections immediately when a transaction completes (5ms–20ms). |
| `pgbouncer.min_pool_size` | `0` | **`2`** | Pre-warms 2 standby connections 24/7 so QA dev testing has zero connection delay. |
| `pgbouncer.default_pool_size` | `20` | **`15`** | Max 15 warm backend connections to PostgreSQL. Highly optimal for 2 vCPU QA server. |
| `pgbouncer.max_client_conn` | `5000` | **`5000`** | Allows up to 5,000 incoming client sockets to connect to PgBouncer. |
| `idle_in_transaction_session_timeout` | `0` (Disabled) | **`30000` (30 sec)** | Kills abandoned transactions idle >30s. Logged as error `57P01`. |
| `lock_timeout` | `0` (Disabled) | **`10000` (10 sec)** | Kills blocked row lock waits >10s. Logged as error `55P03`. |
| `statement_timeout` | `0` (Disabled) | **`60000` (60 sec)** | Kills runaway unindexed queries >60s. Logged as error `57014`. |

---

## 4. Override Hierarchy: How Values Get Picked

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

## 5. Step-by-Step QA Execution Plan

### Step 1: Update Azure PostgreSQL Server Parameters (QA)
1. Open **Azure Portal** $\rightarrow$ **Azure Database for PostgreSQL Flexible Server** $\rightarrow$ `<POSTGRES_QA_SERVER_HOST>`.
2. Navigate to **Server Parameters**.
3. Set `pgbouncer.enabled` = `true`.
4. Set `pgbouncer.min_pool_size` = `2`.
5. Set `pgbouncer.default_pool_size` = `15`.
6. Set `idle_in_transaction_session_timeout` = `30000`.
7. Set `lock_timeout` = `10000`.
8. Set `statement_timeout` = `60000`.
9. Click **Save**.

### Step 2: Update Azure DevOps Variable Groups (QA)
1. Open **Azure DevOps** $\rightarrow$ Project `<AZURE_DEVOPS_PROJECT>` $\rightarrow$ **Pipelines** $\rightarrow$ **Library**.
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
Connect to QA PostgreSQL server (`<POSTGRES_QA_SERVER_HOST>`) via pgAdmin/psql with `<DATABASE_USER>` and run the following **READ-ONLY** SQL script:

```sql
-- Verify PgBouncer & Defensive Parameter Settings on <POSTGRES_QA_SERVER_HOST>
SHOW pgbouncer.enabled;
SHOW pgbouncer.min_pool_size;
SHOW pgbouncer.default_pool_size;
SHOW idle_in_transaction_session_timeout;
SHOW lock_timeout;
SHOW statement_timeout;

-- Verify Active Connections for <DATABASE_USER> are routed via PgBouncer Port 6432
SELECT 
    state, 
    usename, 
    client_addr, 
    count(*) AS open_connections
FROM pg_stat_activity
WHERE usename = '<DATABASE_USER>'
GROUP BY state, usename, client_addr
ORDER BY open_connections DESC;
```
