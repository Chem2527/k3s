# Zenus Digital Portal - OTP & Payment Incident Investigation Report

**Document Name**: `sunil.md`  
**Target Environment**: Azure Container Apps (`rg-purpleplum-prod-eastus`)  
**Azure Tenant**: Zenus Bank International Inc  
**Incident Date**: September 3 - September 4, 2026  

---

## Executive Summary

On September 3, 2026, during manual payment creation via the Back Office portal, an operator experienced an OTP validation error (`"Code cannot be accepted"`). Upon re-submitting, two duplicate transfers of **$1,500,000.00 USD** each (`PAYM-212370` and `PAYM-212372`) were executed successfully in the backend. Additionally, users experienced 24-hour account lockouts during login OTP verification, session dropouts during entity switching, and UI selection issues on sender accounts.

Investigation of production logs via **Azure Log Analytics** (`log-zb-pp-fintech-prod-eastus-001`) and source code in **Azure DevOps** (`ZenusBankInternational`) confirmed this is a **code-side issue** caused by high latency in OTP endpoints, premature OTP code consumption in Redis, and lack of payment idempotency guards.

---

## 1. Production Log Evidence (Log Analytics)

Querying table `ContainerAppConsoleLogs_CL` in workspace `7bc603fe-b97f-4e26-a05d-0cee18e2c133` yielded the following empirical evidence:

### A. Duplicate Transactions
* **Transaction 1 (`PAYM-212370`)**:
  * **Timestamp**: `2026-09-03T17:52:00.147Z`
  * **Payer**: Nexo Capital Inc. (`ID-1015`)
  * **Beneficiary**: Nexo Financial LLC (`ID-134412`)
  * **Amount**: `$1,500,000.00 USD`
  * **Operator**: Biser Manchev
  * **Status**: `PROCESSED` (200 OK via Tuum Payment Provider)
  * **Request ID**: `16600e0b-6b5e-4c32-abaf-1700e66cc277`

* **Transaction 2 (`PAYM-212372`)**:
  * **Timestamp**: `2026-09-03T17:54:02.523Z` (2 minutes 2 seconds later)
  * **Payer**: Nexo Capital Inc. (`ID-1015`)
  * **Beneficiary**: Nexo Financial LLC (`ID-134412`)
  * **Amount**: `$1,500,000.00 USD`
  * **Operator**: Biser Manchev
  * **Status**: `PROCESSED` (200 OK via Tuum Payment Provider)
  * **Request ID**: `bf515757-a2d8-4201-bc22-78a161d11537`

### B. High Latency & HTTP 410 Errors
* **High Endpoint Latency**: Calls to `POST /api/notifications/verifyOtp` took up to **7,378 ms (7.38 seconds)**.
* **HTTP 410 (Gone / Expired)**: Logs recorded `InboundRestRequestFailed -> 410` on re-submitted OTP requests.
* **Sequence of Events**:
  1. Request #1 arrives -> OTP verified in Redis -> marked as consumed (`is_active: false`) -> calls downstream bank API (takes ~7.3s).
  2. Due to the 7-second UI freeze, user re-submits OTP code.
  3. Request #2 arrives -> OTP already marked consumed -> returns **HTTP 410 ("Code cannot be accepted")**.
  4. Meanwhile, Request #1 completes successfully in the background (`PAYM-212370`).
  5. User sees error message, inputs new OTP / clicks submit again -> creates duplicate payment `PAYM-212372`.

---

## 2. Root Cause Summary (In Simple Words)

1. **The 7-Second Freeze**: Processing payments inside the OTP endpoint took 7+ seconds, making users think the submission failed.
2. **False Error Message**: Retried submissions returned HTTP 410 ("Code cannot be accepted") because the 1st request already marked the code as used in Redis.
3. **Missing Payment Idempotency**: The backend processed both submissions because payment endpoints lacked duplicate request locking.
4. **Login Lockout**: Retried login OTPs returned HTTP 410, which the user management service counted as wrong password attempts, triggering 24-hour account locks.

---

## 3. Manual Verification Steps via Azure Portal Console (`portal.azure.com`)

### Step 1: Open Log Analytics Workspaces
1. Go to [portal.azure.com](https://portal.azure.com) and log in as `skumar.purpleplum@zenus.com`.
2. Select Subscription: `sub-zb-prod`.
3. Search for **Log Analytics workspaces** and select `log-zb-pp-fintech-prod-eastus-001`.
4. Click **Logs** in the left menu.

### Step 2: Run KQL Queries

* **Query 1: Duplicate Payments**:
  ```kusto
  ContainerAppConsoleLogs_CL
  | where Log_s contains 'PAYM-212370' or Log_s contains 'PAYM-212372'
  | project TimeGenerated, ContainerAppName_s, Log_s
  | sort by TimeGenerated desc
  ```

* **Query 2: 7-Second OTP Latency**:
  ```kusto
  ContainerAppConsoleLogs_CL
  | where TimeGenerated >= datetime(2026-09-03T12:00:00Z)
  | where Log_s contains 'verifyOtp'
  | project TimeGenerated, ContainerAppName_s, Log_s
  | sort by TimeGenerated desc
  ```

* **Query 3: HTTP 410 Errors**:
  ```kusto
  ContainerAppConsoleLogs_CL
  | where TimeGenerated >= datetime(2026-09-03T12:00:00Z)
  | where Log_s contains 'verifyOtp' and Log_s contains '410'
  | project TimeGenerated, ContainerAppName_s, Log_s
  | sort by TimeGenerated desc
  ```

---


---

## 5. Permanent 100% Code Fix Plan

### Fix 1: Payment Idempotency Guard (Prevents Duplicates)
* **Service**: `fintech_business_management`
* **File**: `src/services/payments/paymentService.ts`
* **Fix**: Implement a Redis Distributed Lock on `X-Request-Id` or request payload hash. If a duplicate payment request arrives within 5 minutes, block execution and return the existing result.

### Fix 2: Decouple OTP Verification from Payment Execution
* **Service**: `fintech_notifications_management`
* **File**: `src/services/otp/otpService.ts`
* **Fix**: Verify OTP in < 0.5 seconds, return a single-use token to the client, and process the payment asynchronously. Exclude HTTP 410 status from incrementing login lockout counters.

### Fix 3: Refresh Token on Entity Switch & Disable UI Double-Click
* **Backend**: `fintech_user_management` (`src/services/userService.ts`) — Issue updated JWT with target `company_id` claims upon entity switch.
* **Frontend**: `fintech_webapp` / `fintech_admin_webapp` — Disable submit button on click (`disabled={isSubmitting}`) to prevent double clicking.
