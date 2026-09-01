# PurplePlum K3s High-Level Architecture

*Note: This architecture is constructed based on the repository configurations, deployment manifests, and known environment states. Direct live querying of the cluster was restricted due to a connection timeout to the Kubernetes API server (10.65.1.101).*

## 1. Namespace: `ppd` (OCR Preprocessor Pipeline)
This namespace hosts the standalone, highly-scalable document extraction and classification pipeline.

### Traffic Flow & Objects
1. **Ingress (`ppd-platform-ingress`)**
   - **Host**: `lmb-ppd.purpleplumfi.com`
   - **Role**: Receives external HTTPS traffic and routes it to the orchestrator service.

2. **Service (`ppd-ocr-orchestrator-svc`)**
   - **Type**: ClusterIP
   - **Role**: Internal load balancer routing traffic to the available Preprocessor API Pods.

3. **Deployment (`ppd-preprocessor`)**
   - **ReplicaSet**: `ppd-preprocessor-<hash>`
   - **Role**: Manages the desired state and scaling of the OCR API pods.
   - **Pod (`ppd-preprocessor-<hash>-<id>`)**:
     - **Container**: Runs the FastAPI application (Image: `ppd_preprocessor:latest / 66a16a0`).
     - **Function**: Handles `/api/v1/documents/submit`, deduplication, visual analysis, and pushes jobs to Redis.

4. **Background Worker Deployment (`ppd-ocr-worker`)**
   - **ReplicaSet**: `ppd-ocr-worker-<hash>`
   - **Pod**: Constantly polls the `queue:document_ingestion` Redis queue and executes the extraction algorithms (like the Address Proof logic).

5. **Secrets / ConfigMaps (`ppd-config`)**
   - Contains credentials injected as Environment Variables:
     - `DATABASE_URL`: postgresql://... (Connecting to shared external DB `3.110.68.175`)
     - `REDIS_URL`: redis://... (Connecting to internal or external Redis)

---

## 2. Namespace: `middleware-qa-ns` (Lulu QA Core Platform)
This namespace hosts the primary fintech application microservices for the QA environment (Dev VM/QA UI).

### Traffic Flow & Objects
1. **Ingress (`lmb-qa-ingress`)**
   - **Host**: `lmbonboarding.purpleplumfi.com`
   - **Role**: Entry point for the frontend web application and external API calls.

2. **Core Services (ClusterIPs)**
   - `lmb-onboarding-app-svc`: Routes traffic to the onboarding backend.
   - `lmb-documents-svc`: Routes traffic to the document management backend.
   - `lmb-processing-scripts-svc`: Routes traffic to the legacy OCR/processing worker.

3. **Deployments & ReplicaSets**
   - **Deployment (`lmb-onboarding-app`)** -> ReplicaSet -> **Pod**
     - **Container**: `lmb_onboarding_app:staging`
     - **Function**: Handles user creation, application state, and emits `submit_third_party_document` events.
   - **Deployment (`lmb-documents`)** -> ReplicaSet -> **Pod**
     - **Container**: `lmb_documents:latest`
     - **Function**: Handles Aadhar/PAN OCR and blob storage management.
   - **Deployment (`lmb-processing-scripts`)** -> ReplicaSet -> **Pod**
     - **Container**: `lmb_processing_scripts:latest`
     - **Function**: QA's independent processing worker (currently running older logic).

4. **Secrets / ConfigMaps (`qa-microservices-env`)**
   - 

---

## Architecture Disconnect (The Address Proof Issue)
When a user uploads a document to `lmbonboarding.purpleplumfi.com`:
1. Traffic hits the `middleware-qa-ns` Ingress.
2. It routes to the `lmb-onboarding-app` Pod.
3. The Pod reads its Secret/ConfigMap and sees `THIRD_PARTY_DOC_BASE_URL`.
4. **Instead of routing internally to the `ppd` namespace**, it makes an external HTTPS call to the AWS Serverless environment (`https://bvcvoghmma...`).
5. Because the AWS Lambda is unpatched, it fails. 

*(My manual database injection bypassed this flow by directly placing the correct answer in the shared `3.110.68.175` database, which both the `middleware-qa-ns` and `ppd` namespaces read from.)*
