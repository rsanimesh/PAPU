Here is a comprehensive architecture document for your review. It has been structured for a senior technical and security audience, integrating all information from the diagram, the meeting recording, and the reference material you provided.

---

### **Databricks AI/ML Platform: Architecture & Security Model**

**Document Version:** 1.0
**Date:** 17 November 2025
**Status:** DRAFT

**Purpose:** This document details the high-level solution architecture for the Databricks AI/ML Platform. It outlines the component structure, data flows, CI/CD processes, and security controls, based on the reference diagram and the security team meeting.

### **Table of Contents**

1.  **Solution Overview**
2.  **Core Architectural Concepts**
3.  **High-Level Workspace Architecture**
4.  **Detailed Component Explanation (By Number)**
5.  **Key Process Flows**
    * 5.1. Data Flow (In, Process, Out)
    * 5.2. CI/CD Pipeline Flow (Path to Production)
6.  **Security & Access Control Model**
7.  **Key Principles, Risks, and Unresolved Points**

---

### **1. Solution Overview**

Databricks is a unified, open analytics platform for building, deploying, sharing, and maintaining enterprise-grade data, analytics, and AI solutions at scale.

This architecture implements a secure, three-tiered Databricks environment to support the AI/ML development lifecycle. It consists of three separate, distinct workspaces: one for analytics and "Lab" development, one for "UAT/Parallel" testing, and a final, locked-down "Prod" workspace.

The solution leverages a federated data model. The AI/ML team will consume approved data from the central Enterprise Data Platform (EDP) and publish its final outputs back to the EDP as a **Consolidated Data Product (CDP)**, which is then consumed by tenant accounts.

### **2. Core Architectural Concepts**

As defined in the solution overview documentation, the platform comprises two building blocks:

* **Control Plane:** The infrastructure used by Databricks to deploy, configure, and manage the platform and services. In the diagram, this is represented by component **(4)**, which generates the dedicated cloud infrastructure.
* **Data Plane:** The customer-owned infrastructure managed in collaboration by Databricks and Barclays. In the diagram, this is the "Compute Plane (Barclays Databricks VPC)" where all workspaces and clusters run. The "Data Plane" section at the bottom of the diagram refers to the persistent storage layer, the EDP Datalake.

### **3. High-Level Workspace Architecture**

The architecture is founded on three separate Databricks workspaces, each with a distinct purpose and security posture:

1.  **Prod Analytics Workspace ("Lab"):** This is the environment for data scientists to experiment, build features, and train models. Users (Data Scientists and the AI/ML team) have read/write access here.
2.  **Prod Parallel Workspace ("UAT"):** This is the User Acceptance Testing (UAT) environment to validate models and code against production-like data and configurations. Access is intended to be read-only for the AI/ML team, with a strong preference from security for no interactive access.
3.  **Prod Workspace ("Production"):** This is the final, locked-down production environment. It has **no interactive access** and is managed exclusively by the "Run The Bank" (RTB) team via automated CI/CD pipelines, triggered by a Change Request (CR).

### **4. Detailed Component Explanation (By Number)**

1.  **Unity Catalog (Sync):** This is the data entry point for the Databricks ecosystem. It syncs with the EDP Datalake (3) to allow the "Prod Analytics" workspace to discover and access approved bank data from the central catalog.
2.  **Databricks Bucket (Prod Analytics):** A **perishable, temporary S3 bucket** located in the federated BUK AIL AWS account. It holds interim data (e.g., feature engineering outputs) during model development. This bucket is subject to an **automated 90-day retention policy**.
3.  **EDP Datalake (Central Catalog):** The bank's central data marketplace, aligning with the "Current EDP Strategy". It serves as both the **source** of data (via #1) and the **destination** for the final, registered Consolidated Data Product (CDP).
4.  **Databricks Dedicated Cloud Infrastructure:** The Databricks Control Plane component that manages and orchestrates the three separate workspaces within the Barclays VPC.
5.  **Databricks Bucket (Prod Parallel):** The temporary, non-production S3 bucket for the UAT/Parallel workspace.
6.  **GitLab (Feature/Dev Branch):** The code repository for the "Prod Analytics (Lab)" workspace. Data scientists commit and push development code here.
7.  **GitLab (UAT/Release Branch):** The code repository for the "Prod Parallel (UAT)" workspace. Code is promoted from the dev branch (6) to this branch to trigger UAT deployment.
8.  **Nexus (Model Registry):** The central, version-controlled repository for **model binaries**. When a model is finalized in the Lab, its binary file is published here.
9.  **GitLab Runner (UAT):** The CI/CD pipeline that automates the deployment of code from the UAT branch (7) and the model from Nexus (8) into the "Prod Parallel Workspace" for testing.
10. **GitLab Runner (Prod):** The production deployment pipeline. This is triggered **only after a Change Request (CR) is approved** and is executed by the **RTB team**.
11. **Model Execution / Monitoring (Prod):** The model and code running in the secure "Prod Workspace". This environment has **no human interactive access**.
12. **Databricks Bucket (Prod):** The S3 bucket for the production workspace. The final model output (CDP) is stored here before being formally synced and registered with the EDP Central Catalog (3).
13. **Tenant Account (e.g., Martech):** The end-consumer of the data product. They **do not** access the BUK AIL AWS accounts directly. They securely access the final inference data (CDP) from the **EDP Central Catalog (3)**.

### **5. Key Process Flows**

#### **5.1. Data Flow (In, Process, Out)**

* **Data In:** Data is sourced from the **EDP Datalake (3)** via the **Unity Catalog (1)** into the "Prod Analytics" workspace.
* **Data in Process:** Interim development outputs are stored in the perishable **Lab Bucket (2)**.
* **Data Out (Production):** The final output (CDP) is generated in the "Prod Workspace" (11), stored in the **Prod Bucket (12)**, and formally registered in the **EDP Central Catalog (3)**.
* **Data Consumption:** The **Tenant Account (13)** accesses the CDP from the **Central Catalog (3)**.

#### **5.2. CI/CD Pipeline Flow (Path to Production)**

* **Step 1 (Development):** Code is written in the Lab, pushed to the **GitLab Dev Branch (6)**. The final model binary is published to **Nexus (8)**.
* **Step 2 (Testing):** Code is merged to the **UAT Branch (7)**. The **UAT Runner (9)** automatically deploys the code (7) and model (8) to the "Prod Parallel Workspace" for validation.
* **Step 3 (Production):** A **Change Request (CR) is approved**. The **RTB team** executes the **Prod Runner (10)**. This runner deploys the validated code (from a release tag) and model (8) to the "Prod Workspace" (11).

### **6. Security & Access Control Model**

* **Prod Analytics (Lab) Workspace:**
    * **Personas:** Data Scientists, AI/ML Engineers.
    * **Access:** Read/Write.
    * **Purpose:** Experimentation, development.
    * **Controls:** Data sourced from Unity Catalog (1). Perishable storage (2). No data download.

* **Prod Parallel (UAT) Workspace:**
    * **Personas:** AI/ML Team (for validation), CI/CD Service Principals.
    * **Access:** Read-Only (Team) / No Interactive Access (Security Preference).
    * **Purpose:** UAT, integration testing.

* **Prod (Production) Workspace:**
    * **Personas:** RTB Team (via automation), CI/CD Service Principals.
    * **Access:** **No Interactive Access**.
    * **Purpose:** Serving production model.
    * **Controls:** "Break glass" environment. All deployments are fully automated, require a CR, and are executed by the RTB team.

### **7. Key Principles, Risks, and Unresolved Points**

#### **Key Security Principles**
* **Federated Data Model:** The team consumes from the Central Catalog and publishes its output (CDP) back to the Central Catalog, all from its own federated AWS account.
* **Perishable Lab Storage:** The storage (Bucket 2) used in the "Prod Analytics" (Lab) environment is strictly temporary and must have automated retention to prevent it from becoming a source of unmanaged production data.
* **No Data Download:** To protect data lineage and meet regulatory obligations (e.g., GDPR), the capability to download data from the Databricks environment to a local machine is prohibited. All analysis must be performed within the confines of the environment.
* **Production Lockdown:** The Production workspace (11) is completely locked from human interactive access, with all changes managed by an automated, CR-based process run by the RTB team.

#### **Unresolved Points for Further Discussion**
* **Access to Prod Parallel (UAT) Workspace:** There is an open point of discussion regarding the UAT workspace.
    * **Security's Stance:** Prefers this environment to have no interactive access, mirroring production.
    * **AI/ML Team's Use Case:** A requirement was raised for limited, configuration-only access to manage tasks like model throttling for canary or blue-green deployments.
    * **Next Step:** Further discussion is needed to find a granular solution that meets this use case without compromising the security posture.

#### **Active Security Workstreams**
* The security team is actively working to validate and harden the Databricks environment against data exfiltration vectors, such as unauthorized CLI or IDE-based downloads.
* A pen test of the BUK AIL AWS account is planned or in progress.
