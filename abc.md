# Zenus Digital Portal - OTP & Payment Incident Investigation Report

**Document Name**: `sunil.md`  
**Target Environment**: Azure Container Apps (`rg-purpleplum-prod-eastus`)  
**Azure Tenant**: Zenus Bank International Inc  
**Incident Date**: September 3 - September 4, 2026  

---

## Executive Summary

On September 3, 2026, during manual payment creation via the Back Office portal, an operator experienced an OTP validation error (`"Code cannot be accepted"`). Upon re-submitting, two duplicate transfers of **$1,500,000.00 USD** each (`PAYM-212370` and `PAYM-212372`) were executed successfully in the backend. Additionally, users experienced 24-hour account lockouts during login OTP verification, session dropouts during entity switching, and UI selection issues on sender accounts.

Investigation of live production logs via **Azure Log Analytics** (`log-zb-pp-fintech-prod-eastus-001`), **Application Insights** (`insight-zb-purpleplum-prod-eastus`), and source code in **Azure DevOps** (`ZenusBankInternational`) confirmed this is a **code-side issue** caused by high latency in OTP endpoints, PostgreSQL database transaction row locking, premature OTP code consumption in Redis, and lack of payment idempotency guards.

---

## 1. Why the 11-Second Database Lock Happened (Exact Root Cause Found in Code)

In `fintech_notifications_management` (`src/services/otp/otpService.ts`):

1. **Uncommitted Database Transaction Held During External API Calls**:
   * **Line 719**: The code opens an explicit database transaction: `const tranx = await connection.transaction();`.
   * **Line 735**: It executes a database update on `otps`: `await OtpModel.query(tranx).update(queryParams)...`. This places an **exclusive PostgreSQL row lock** on the `otps` table row.
   * **Lines 764 & 815–856**: While `tranx` is still open and uncommitted, the code makes multiple **outbound HTTP REST API calls** (`sendOtpLoginResponse`, `checkForApproval`, and Tuum bank payment creation calls).
   * **Line 857**: `await tranx.commit();` is not executed until after all external REST calls finish.

2. **The Resulting PostgreSQL Lock Cascade**:
   * Because calling external bank APIs takes 7 to 11 seconds, the PostgreSQL transaction `tranx` stays OPEN for **11.7 seconds**, holding exclusive row locks on `otps` and `users` tables.
   * Any concurrent HTTP request trying to read or verify an OTP gets **BLOCKED by PostgreSQL** waiting for `tranx.commit()` to release the lock.
   * This matches the Application Insights telemetry, which recorded database queries hanging for **110 seconds** (`pg.query:SELECT`) and **929 seconds** (`pg.query:UPDATE`) on `zb-psql-pp-prd-eastus.postgres.database.azure.com`.

---

## 2. What Is in the Existing Code (`C:\Users\nxtge\Downloads\zenus`)

* **`fintech_notifications_management` (`src/services/otp/otpService.ts`)**:
  * Line 674 immediately sets `is_active = false` in Redis as soon as the OTP code is read.
  * If a retried request hits `verifyOTP()`, line 627 (`.where("otps.is_active", "true")`) fails to find the code and line 631 throws **HTTP 410 ("Code cannot be accepted")**.
  * On rejected OTPs, line 690 increments the user's failed login attempt counter, triggering the 24-hour account lock.

* **`fintech_business_management` (`src/services/payments/paymentService.ts`)**:
  * Payment endpoints (`initialiseSelfTransfer`, `initialisePayment`, `confirmPayment`) do **not** check for duplicate request headers or Redis idempotency keys before sending transactions to Tuum (`payment-api.zenus.tuumplatform.com`).

* **`fintech_user_management` (`src/services/userService.ts`)**:
  * When switching company entities, the backend updates `company_id` in DB but does **not** issue a renewed JWT token containing the new `company_id` claims to the frontend.

---

## 3. What Is in the Logs & Application Insights Telemetry

### A. Log Analytics (`log-zb-pp-fintech-prod-eastus-001`)
* **Duplicate Transactions**:
  * **Transaction 1 (`PAYM-212370`)**: Inserted at `2026-09-03T17:52:00.147Z` | Amount: `$1,500,000.00 USD` | Payer: Nexo Capital Inc. (`ID-1015`) | Beneficiary: Nexo Financial LLC (`ID-134412`) | Operator: Biser Manchev | Request ID: `16600e0b-6b5e-4c32-abaf-1700e66cc277`.
  * **Transaction 2 (`PAYM-212372`)**: Inserted at `2026-09-03T17:54:02.523Z` (2 minutes 2 seconds later) | Amount: `$1,500,000.00 USD` | Payer: Nexo Capital Inc. (`ID-1015`) | Beneficiary: Nexo Financial LLC (`ID-134412`) | Operator: Biser Manchev | Request ID: `bf515757-a2d8-4201-bc22-78a161d11537`.

* **Endpoint Latency**:
  * `POST /api/notifications/verifyOtp` response times recorded up to **11,758 ms (11.75 seconds)**.

### B. Application Insights (`insight-zb-purpleplum-prod-eastus`)
* **PostgreSQL Database Query Locks (`zb-psql-pp-prd-eastus`)**:
  * Database `dependencies` telemetry recorded `SELECT` queries taking up to **110,622 ms (110 seconds)** and `UPDATE` queries taking up to **929,360 ms (~15 minutes)**.
* **PostgreSQL Duplicate Key Constraint Errors (Code 23505)**:
  * `exceptions` telemetry logged multiple `PostgreSQL error (code: 23505)` errors (`unique_violation`) caused by concurrent retries writing identical keys.
* **HTTP 401 Unauthorized Exceptions**:
  * `exceptions` telemetry logged `AxiosError 401 Unauthorized` on `/dashboard` during company entity switching.

---

## 4. What the Exact Issue Is (In Simple Words)

1. **The 11-Second Database Lock**: An uncommitted PostgreSQL transaction held open during external API calls caused `/verifyOtp` to freeze for **11.7 seconds**.
2. **The Double Click**: Thinking the click failed, the user clicked "Submit" a second time.
3. **The False Error**: The 1st click had already marked the OTP as "used" in Redis. When the 2nd click arrived, the server returned **HTTP 410 ("Code cannot be accepted")**.
4. **The Duplicate Payment**: The 1st click was still processing in the background and **successfully sent $1.5M** (`PAYM-212370`). Seeing the error message, the user tried again 2 minutes later, and **a second $1.5M payment went through** (`PAYM-212372`) because the backend lacked an Idempotency Lock.

---

## 5. The Exact 100% Permanent Solution (In Simple Words)

### Solution 1: Fix Database Transaction Scope (Eliminates the 11s Lock)
* In `fintech_notifications_management` (`src/services/otp/otpService.ts`):
  * **Commit the DB transaction `tranx` immediately** after updating the `otps` record (Line 736). Do **NOT** keep `tranx` open while making outbound HTTP REST API calls (`sendOtpLoginResponse`, Tuum calls).

### Solution 2: Add Payment Idempotency Lock (Prevents Duplicates)
* In `fintech_business_management`, add a **5-minute Redis Lock** on the payment request ID or `(user_id + amount + beneficiary_id)`.
* If a duplicate payment request arrives, **block the 2nd payment** and immediately return the result of the 1st transaction.

### Solution 3: Instant OTP Verification (< 0.5s)
* Separate OTP checking from payment processing.
* Verify the OTP code instantly in **< 0.5 seconds**, return an authorized single-use token to the client, and run the bank transfer asynchronously.
* Do not count HTTP 410 (already consumed OTP) as a failed password attempt to prevent 24-hour account locks.

### Solution 4: Fix Entity Switch Token & UI Button
* In `fintech_user_management`, issue a fresh JWT token with updated `company_id` claims when switching entities so users stay logged in.
* In `fintech_webapp`, immediately disable the "Submit" button (`disabled={isSubmitting}`) upon click to prevent double-clicking.

---

## 6. Manual KQL Queries for Azure Portal UI (`portal.azure.com`)

Open **`portal.azure.com`** -> **`Application Insights`** -> **`insight-zb-purpleplum-prod-eastus`** -> **`Logs`**:

### Query 1: Filter Only Slow or Failed Requests
```kusto
requests
| where timestamp >= datetime(2026-09-03T12:00:00Z)
| where name contains "verifyOtp" or name contains "payment"
| where success == false or resultCode != "200" or duration > 5000
| project timestamp, name, resultCode, success, duration, url
| sort by duration desc
```

### Query 2: View PostgreSQL Database Query Locks
```kusto
dependencies
| where timestamp >= datetime(2026-09-03T12:00:00Z)
| where target contains "postgres" and duration > 5000
| project timestamp, target, name, duration, resultCode, success
| sort by duration desc
```

### Query 3: View Database Constraint & 401 Exceptions
```kusto
exceptions
| where timestamp >= datetime(2026-09-03T12:00:00Z)
| project timestamp, type, outerMessage, operation_Name
| sort by timestamp desc
```
