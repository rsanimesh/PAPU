# Barclays BUK AIML – Databricks AI/ML Platform Architecture  
**High-Level Design (HLD) – Version 1.1**  
**Date: 17 November 2025**  
**Author: AI/ML Engineering Team (with Security & EDP alignment)**  

## 1. Architecture Overview

The proposed architecture migrates Barclays BUK AIML workloads from the existing AWS SageMaker construct to Barclays’ strategic Databricks platform while maintaining (and strengthening) security, governance, and data lineage requirements.

The solution adopts **three physically separate Databricks workspaces** inside the Barclays Databricks VPC (customer Data Plane):

| Workspace             | Purpose                          | Equivalent Legacy Environment | Expected Human Access                  |
|-----------------------|----------------------------------|-------------------------------|-----------------------------------------|
| Prod Analytics        | Development / Data Science Lab   | Build / Dev                   | Data Scientists + AIML Engineering (read/write) |
| Prod Parallel         | Integration / UAT / Pre-Prod     | Test                          | **Proposed**: Limited privileged access for configuration-only changes (throttling, canary, blue-green). **Security preference**: No interactive logins – service principals only. **Status**: Subject to final security & EDP approval |
| Prod                  | Production (Run-The-Bank)        | Prod                          | None – service principals only, deployments via RTB Change Request |

All data access is governed exclusively through **Unity Catalog ↔ EDP Central Catalog** synchronisation (the single blue line in the diagram). Direct cross-account or raw-bucket access is prohibited.

The AIML account operates as a **federated producer/consumer** under the EDP data mesh: it consumes Foundation/Consolidated Data Products and publishes its own Consolidated Data Products (model inferences, feature stores) back to the Central Catalog.

## 2. Numbered Component Explanation (as per diagram)

| # | Component                              | Description                                                                                                                                            | Key Characteristics / Constraints                                                                                                             |
| --- |----------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| 1   | Unity Catalog ↔ EDP Central Catalog Sync | Bi-directional governed interface to the bank’s data marketplace.                                                                                    | Only approved Foundation, Consolidated or Business Data Products are exposed. Exceptions for raw buckets require explicit CTO/CDAO approval. |
| 2   | Databricks Bucket (Prod Analytics / Lab) | Temporary storage in the BUK AIML federated AWS account for feature engineering outputs, training datasets, and interim artefacts.                 | • “Data in flight” – perishable<br>• Automated retention policy mandatory (current analytics precedent: 90 days)<br>• Must be versioned and auto-deleted after model promotion<br>• No permanent production data |
| 3   | On-Prem / Data Sync → EDP Datalake → Prod EDP Bucket → BUK AIML Feature Store | Source data ingestion path into the EDP ecosystem.                                                                                                     | Not in AIML scope – shown for context only.                                                                                                   |
| 4   | Databricks Dedicated Cloud Infrastructure (Control Plane orchestration) | Databricks mechanism that provisions and manages the three separate workspaces inside the Barclays VPC.                                           | Customer-managed portion of Databricks Control Plane.                                                                                         |
| 5   | GitLab (Dev / Feature Branch)          | Source code & notebook repository for the Lab workspace.                                                                                              | Standard GitLab branching model.                                                                                                              |
| 6   | GitLab (UAT / Release Branch)          | Branch used for promotion to Prod Parallel workspace.                                                                                                  | Triggers CI/CD pipeline for integration testing.                                                                                              |
| 7   | Integration Testing / UAT             | Execution environment inside Prod Parallel workspace.                                                                                                  | Model binary pulled from Nexus, code from GitLab release tag.                                                                                 |
| 8   | Nexus Model Registry                   | Central artefact repository for versioned model binaries (no training data).                                                                           | Only serialized model files – no PII or training datasets.                                                                                    |
| 9   | GitLab Runner (UAT)                    | CI/CD runner that deploys approved code + model to Prod Parallel workspace.                                                                            | Automated; may require configuration parameters for throttling / canary.                                                                     |
| 10  | GitLab Runner (Prod)                   | Production deployment pipeline – triggered only after approved Change Request (CR).                                                                   | Executed by RTB team only.                                                                                                                    |
| 11  | Model Execution & Monitoring (Prod)    | Production model serving and monitoring inside the Prod workspace.                                                                                    | No interactive human access – service principals only.                                                                                       |
| 12  | Databricks Bucket (Prod)               | Production bucket that receives final inference output.                                                                                                | Output registered as official Consolidated Data Product (CDP) and synced to EDP Central Catalog. Permanent retention (governed by EDP policy). |
| 13  | Tenant Account (e.g., Martech, Complaints, etc.) | Downstream consumer domains.                                                                                                                           | Consumers access AIML CDP exclusively via EDP Central Catalog – never direct access to AIML buckets or accounts.                               |

## 3. Data Flow Summary

1. **Inbound**: Data Scientists query approved data via Unity Catalog (1) → materialise interim data in perishable Lab bucket (2).  
2. **Development**: Feature engineering, model training, experimentation occur entirely within Prod Analytics workspace.  
3. **Promotion**: Finalised model binary → Nexus (8). Code → GitLab release tag.  
4. **UAT**: Automated (or limited config-only) deployment to Prod Parallel for integration / performance / canary testing.  
5. **Production**: RTB-triggered deployment to Prod workspace → inference → output written to Prod bucket (12) → registered as CDP in EDP Central Catalog.  
6. **Consumption**: Tenant domains (13) consume the CDP via the marketplace – no cross-account access.

## 4. Security & Governance Principles (Agreed in Security Review)

| Principle                                   | Detail                                                                                                             | Status      |
|---------------------------------------------|--------------------------------------------------------------------------------------------------------------------|-------------|
| Federated EDP Model                         | AIML is both consumer and producer of CDPs via Central Catalog                                                    | Agreed      |
| Perishable Non-Prod Storage                 | Buckets 2 (and any Parallel equivalent) must auto-expire; no permanent production data outside EDP                 | Agreed      |
| No Local Data Download / Exfiltration       | Databricks CLI / %fs magic / emulator download capability is an open risk – must be disabled or blocked           | Open – high risk; working with Databricks & CSO |
| Zero Interactive Access in Prod             | Prod workspace: service principals only                                                                            | Agreed      |
| Prod Parallel Access                        | Security preference: no human logins. AIML requires limited config-only access for throttling / canary deployments | **Open – requires further discussion & granular controls** |
| All Data Access via Unity Catalog          | No bypassing of Central Catalog except pre-approved raw-bucket exceptions                                         | Agreed      |
| GDPR / Right-to-be-Forgotten Compliance    | Training datasets in Lab bucket must be deletable via automated retention (cannot be scattered to laptops)         | Agreed      |

## 5. Open Risks & Next Actions

| Risk / Item                                      | Impact                        | Owner                  | Target Resolution |
|--------------------------------------------------|-------------------------------|------------------------|-------------------|
| Databricks data exfiltration via CLI / %fs       | High – potential GDPR breach  | Security + Databricks  | Dec 2025          |
| Prod Parallel workspace & access model approval | High – may force single-workspace model | CSO / EDP / AIML       | Nov 2025          |
| Exact retention period for Lab bucket (90 days vs shorter) | Medium                    | EDP + AIML             | Nov 2025          |
| Granular privileged access mechanism for Prod Parallel configuration changes | Medium                | Security + AIML        | Dec 2025          |

This document aligns with the security review held on 14 November 2025 and incorporates all agreed principles. Further refinement will occur once the above open items are closed.

**Prepared by**: BUK AIML Engineering  
**Reviewed & Aligned**: Barclays Security, EDP Data Platform, RTB (pending final sign-off on Prod Parallel)
