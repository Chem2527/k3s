# Zenus QA Environment Analysis – Database & Infrastructure Migration Guide

This document is a concise guide covering why our containers failed to scale during load, why PgBouncer is needed, the current QA database limitation, and the step-by-step fix.

---

## Questions Covered in This Document

1. [Why didn't containers scale up even when CPU reached 100%-410%?](#1-why-containers-did-not-scale-up-keda--http-scaler-limitation)
2. [Why can CPU hit 400% with fewer than 10 concurrent requests?](#2-scenarios-where-cpu-spikes-with-low-http-requests)
3. [Why doesn't our QA database currently support PgBouncer, and what needs to change?](#3-qa-database-limitation--required-sku-upgrade)
4. [What happens if we get 42 or 82 requests with our current setup?](#4-current-connection-behavior-without-pgbouncer)
5. [How does PgBouncer solve connection exhaustion, and why default_pool_size = 25?](#5-pgbouncer-solution--connection-math)
6. [Which configuration takes priority: Azure DevOps or Container App settings?](#6-configuration-override-hierarchy)
7. [What are the step-by-step actions to execute in QA?](#7-step-by-step-execution-checklist)

---

## 1. Why Containers Did Not Scale Up (KEDA / HTTP Scaler Limitation)

Azure Container Apps uses **KEDA** for autoscaling. Across all our microservices, only one scale rule is configured:
```json
"rules": [
  {
    "name": "http-scaler",
    "http": {
      "metadata": {
        "concurrentRequests": "10"
      }
    }
  }
]
```

* **The Problem**: The HTTP scaler only looks at active in-flight HTTP requests passing through Envoy reverse proxy at that exact second.
* It is **completely blind to CPU and Memory**. Even when a container hits 400% CPU or 95% RAM, if there are fewer than 10 concurrent HTTP requests, KEDA decides: **Do not scale**.
* Because there are **zero CPU rules** and **zero Memory rulesJ*, the container stays stuck at 1 replica.

---

## 2. Scenarios Where CPU Spikes with Low HTTP Requests

A single container easily reaches 100%-400% CPU with just 1 to 3 concurrent requests:

1. **Heavy Reports & Document Exports**: Generating a single large PDF or Excel statement is just **1 HTTP request** (`concurrentRequests = 1 < 10`), but parsing tens of thousands of records locks the Node.js event loop and saturates available cores.
2. **Cryptographic & Token Operations**: RSA signature verification (`JPM_PRIVATE_KEYP), password hashing, and payload encryption execute heavy math on worker threads. 2 to 3 simultaneous requests max out CPU capacity.
3. **Slow DB Queries & Memory Pressure**: If queries stall on database locks, memory buffers accumulate, triggering continuous V8 garbage collection cycles that spike CPU.

3## Telemetry Evidence (Azure Portal):

#### Figure 1: zb-qa-pp-user-management-001 (CPU > 100% with 0 Scale-Out)
CPU repeatedly spikes past 100%(peaking at 102%), while replica count stays flat at 1 replica because concurrent requests remained below 10:

<img width="1917" height="932" alt="image" src="https://github.com/user-attachments/assets/65226076-a5a3-41fb-acc7-f6084aa33d9e" />

#### Figure 2: zb-qa-pp-super-admin-001 (CPU > 400% with 0 Scale-Out)
CPU hits a 410% multi-core burst, while replicas stay locked at 1. The HTTP scaler never triggered a second replica:

<img width="975" height="449" alt="image" src="https://github.com/user-attachments/assets/16fe1702-40cb-4f94-895c-94636b999cbf" />

> **Production Risk**: In PROD, microservices have this exact same single HTTP rule. A heavy request will choke replicas, freezing user screens with HTTP 504 timeouts.

---

## 3. QA Database Limitation & Required SKU Upgrade

When checking the live Azure PostgreSQL servers:
* **PROD (`zb-psql-pp-prd-eastus`)**: Runs on **GeneralPurpose `Standard_D8ds_v5`** (8 vCPU / 32 GB RAM). Parameter `pgbouncer.enabled` is supported and ready to turn on.
* **QA (`zb-psql-pp-qa-eastus-001`)**: Runs on **Burstable `Standard_B1ms`** (1 vCPU / 2 GB RAM).

### The Blocker in QA:
When attempting to enable PgBouncer on QA, Azure CLI returns: 

```	ext
ERROR: (ServerConfigurationNotAllowed) Server parameter 'pgbouncer.enabled' isn't supported in server 'zb-psql-pp-qa-eastus-001'.
```
Azure Flexible Server **does not support built-in PgBouncer on Burstable tier (`B1ms`)j*.

### The Solution:
* Upgrade the QA database compute tier from **Burstable `Standard_B1ms`** to a **General Purpose tier with at least 2 vCPUs** (such as **`Standard_D2ds_v5`**).
* Once upgraded, Azure immediately unlocks `pgbouncer.enabled` and pool settings.

---

## 4. Current Connection Behavior Without PgBouncer

In Azure DevOps variable groups, 5 services have `DB_MAX_CLIENTS = 20` and 1 service has `DB_MAX_CLIENTS = 40`.

### What happens when 42 or 82 requests arrive?
1. **At 2 Replicas**:
   * Total connection pool = 40 (and 80 for documents).
   * If 42 requests hit the 5 services, 40 are served; requests #41 and #42 wait in Knex's in-memory queue inside Node.js (`acquireConnectionTimeout`, default 60s).
   * If queries take time, Knex throws: `Knex: Timeout acquiring a connection. The pool is probably full`, causing user requests to fail.
2. **Under Autoscaling (10 Replicas each)**:
   * Across all 6 services: (5 * 10 * 20) + (1 * 10 * 40) = **1,400 direct connections**.
   * PostgreSQL has `max_connections = 500`.
   * Connection #501 is rejected with:
     ``	ext
     FATAL: sorry, too many clients already
     ```
     This causes an immediate system-wide portal outage.

---

## 5. PgBouncer Solution & Connection Math

With PgBouncer, applications connect to **Port 6432**. PgBouncer queues requests and shares a small set of warm database connections.

### Pool Sizing: Why `default_pool_size = 25`?
* **Base Core Formula**: `connections = ((core_count * 2) + effective_spindle_count)`. For 2 vCPUs, base active connections = 6.
* **Why 25 for QA**: Setting `pgbouncer.default_pool_size = 25` (provides a generous buffer for simultaneous QA automated regression suites, manual tester actions, and batch queries without placing excessive pressure on the database engine).
* **Standby Connections**: `pgbouncer.min_pool_size = 2` (keeps 2 warm connections open 24/7 so queries don't suffer connection-handshake latency).

### Why `DB_MAX_CLIENTS = 3` in Variable Groups?
* OLTP database queries finish in **5 ms**.
* 1 socket runs: 1,000 ms / 5 ms = **200 queries/sec**.
* 3 sockets per container handle: 3 * 200 = **600 queries/sec**.
* 3 sockets give plenty of throughput while keeping overall socket count small and clean.

---

## 6. Configuration Override Hierarchy

```	ext
Azure DevOps Variable Group (DB_PORT=6432, DB_MAX_CLIENTS=3)
               |
               v
Container App Runtime Environment Variables (process.env)
   [Overrides any hardcoded defaults in application code]
               |
               v
Application Driver (Knex / pg pool)
               |
               v
PgBouncer Pooler (Port 6432)
               |  [Multiplexes down to default_pool_size: 25 QA / 50 PROD]
               v
PostgreSQL Database (Port 5432)
```

---

## 7. Step-by-Step Execution Checklist

### Step 1: Upgrade QA Database Compute Tier
1. Open Azure Portal -> `zb-psql-pp-qa-eastus-001`.
2. Under **Compute + Storage**, change tier from **Burstable `Standard_B1ms`** to **General Purpose `Standard_D2ds_v5`** (2 vCPUs / 8 GB RAM).
3. Save and wait for restart to finish.

### Step 2: Configure PostgreSQL Server Parameters
In **Server Parameters** on the database server, configure only the connection pool parameters (leaving query/lock timeouts at defaults to prevent premature query cancellations):
* ``pgbouncer.enabled` = `true`
* `pgbouncer.min_pool_size` = `2`
* `pgbouncer.default_pool_size` = `25` *(Provides extra connection buffer across all 6 services during simultaneous QA testing)*

### Step 3: Add CPU & Memory Rules in Container Apps
In Azure Container Apps for each service under **Scale**, add:
* **CPU Scaler**: Custom metric `cpu`, utilization `70%.
* **Memory Scaler**: Custom metric `memory`, utilization `75%`.
* Keep existing `http-scaler` (`concurrentRequests = 10`).

### Step 4: Update Azure DevOps QA Variable Groups
In Azure DevOps Library, update the 6 QA variable groups (`ZB-FintechUserManagment-QA`, etc.):
 * `DB_PORT`: `6432`
 * `DB_MAX_CLIENTS`: `3`

> **Note**: Keep `DB_PORT = 5432` for migrations and processing scripts because DDL changes like `ALTER TABLE` require direct port.

### Step 5: Deploy and Verify (Read-Only)
Run this read-only query on PostgreSQL to confirm client connections are routed through port 6432:
```sql
SELECT state, usename, client_addr, count(*) AS open_connections
FROM pg_stat_activity
GROUP BY state, usename, client_addr
ORDER BY open_connections DESC;
```
