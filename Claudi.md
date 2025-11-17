CLauid

# Barclays BUK AIML – Databricks AI/ML Platform Architecture
**High-Level Design Document**  
**Version:** 1.2 DRAFT  
**Date:** 17 November 2025  
**Classification:** Internal - Confidential  
**Status:** Awaiting Security & EDP Final Sign-off

---

## Document Control

| Version | Date | Author | Change Description | Approvers |
|---------|------|--------|-------------------|-----------|
| 1.0 | 14-Nov-2025 | AIML Engineering | Initial draft post security review | Pending |
| 1.1 | 15-Nov-2025 | AIML Engineering | Incorporated security feedback | Pending |
| 1.2 | 17-Nov-2025 | AIML Engineering | Combined architecture details + risk matrix | **Pending CSO, EDP, RTB** |

**Review Meeting Reference:** Security Architecture Review - 14 November 2025, 3:07 PM  
**Key Participants:** Salman (Security), Shantanu (AIML), Swapnil (EDP), Primal, Rahul

---

## Executive Summary

This document details the migration of Barclays BUK AIML workloads from the existing AWS SageMaker environment to Barclays' strategic Databricks platform. The solution implements a **three-workspace security model** with enhanced governance controls while maintaining compliance with banking regulations, GDPR requirements, and the Enterprise Data Platform (EDP) federated data mesh strategy.

**Key Architecture Principles:**
- **Three physically separate Databricks workspaces** with distinct security postures
- **Zero data download capability** to prevent exfiltration and maintain data lineage
- **Federated data producer/consumer model** via EDP Central Catalog
- **Perishable non-production storage** with automated retention policies
- **Production lockdown** with no interactive human access

**Critical Dependencies:**
- Resolution of Databricks CLI/exfiltration security risk (**HIGH PRIORITY**)
- Finalization of Prod Parallel workspace access model
- EDP Central Catalog integration readiness
- Network security controls validation

---

## Table of Contents

1. [Solution Overview](#1-solution-overview)
2. [Core Architectural Concepts](#2-core-architectural-concepts)
3. [Workspace Architecture](#3-workspace-architecture)
4. [Detailed Component Explanation](#4-detailed-component-explanation)
5. [Network Architecture & Security](#5-network-architecture--security)
6. [Data Flow Patterns](#6-data-flow-patterns)
7. [CI/CD Pipeline Flow](#7-cicd-pipeline-flow)
8. [Security & Access Control Model](#8-security--access-control-model)
9. [Data Classification & Handling](#9-data-classification--handling)
10. [Encryption & Data Protection](#10-encryption--data-protection)
11. [Key Principles, Risks & Open Items](#11-key-principles-risks--open-items)
12. [Appendices](#12-appendices)

---

## 1. Solution Overview

### 1.1 Business Context

Databricks is Barclays' strategic unified analytics platform for building, deploying, and maintaining enterprise-grade data, analytics, and AI solutions at scale. This migration from AWS SageMaker to Databricks aligns with the bank's **EDP Strategy** and provides enhanced governance, lineage tracking, and security controls.

The BUK AIML team operates as a **federated producer and consumer** within the EDP data mesh:
- **Consumer:** Foundation and Consolidated Data Products from the EDP Central Catalog
- **Producer:** Consolidated Data Products (CDPs) - model inferences, feature stores, and ML outputs

### 1.2 Architecture Philosophy

The solution implements a **defense-in-depth security model** with:
- **Workspace isolation:** Three separate workspaces with distinct VPCs and security boundaries
- **Data governance:** All data access exclusively through Unity Catalog ↔ EDP Central Catalog sync
- **Perishable storage:** Temporary data in non-production environments with automated retention
- **Zero trust production:** No interactive access to production environment
- **Audit & lineage:** Complete traceability from source to consumption

### 1.3 Key Stakeholders

| Stakeholder Group | Role | Responsibility |
|-------------------|------|----------------|
| AIML Engineering Team | Solution Owner | Development, model training, feature engineering |
| Data Scientists | Primary Users | Model experimentation, algorithm development |
| RTB (Run The Bank) Team | Production Operations | Production deployments via CR process |
| EDP Data Platform Team | Data Governance | Central Catalog management, Unity Catalog sync |
| Security (CSO) | Risk & Compliance | Access controls, security architecture approval |
| Tenant Accounts (MDP, etc.) | Data Consumers | Consume AIML CDPs for downstream applications |

---

## 2. Core Architectural Concepts

### 2.1 Databricks Platform Components

The Databricks platform consists of two fundamental layers:

#### **Control Plane**
- **Definition:** Infrastructure managed by Databricks to deploy, configure, and orchestrate platform services
- **Location:** Databricks-managed cloud environment
- **Responsibilities:** 
  - Workspace provisioning and management
  - Cluster orchestration
  - Job scheduling
  - Unity Catalog metadata management
- **Diagram Reference:** Component **(4)** - Databricks Dedicated Cloud Infrastructure

#### **Data Plane**
- **Definition:** Customer-owned infrastructure within Barclays VPC where workloads execute
- **Location:** Barclays Databricks VPC (Customer-managed AWS account)
- **Components:**
  - Three separate workspaces (Prod Analytics, Prod Parallel, Prod)
  - Compute clusters (Spark, MLflow)
  - Temporary storage buckets
  - Unity Catalog clients
- **Diagram Reference:** "Compute Plane (Barclays Databricks VPC)" section

#### **Storage Layer**
- **Definition:** Persistent data storage managed by EDP
- **Location:** EDP Datalake (Central AWS accounts)
- **Purpose:** 
  - Source of truth for all bank data
  - Foundation, Consolidated, and Business Data Products
  - Final destination for AIML CDP outputs
- **Diagram Reference:** "Data Plane" section at bottom of diagram

### 2.2 Workspace vs Tenancy Model

**Critical Architectural Decision:** Following extensive discussion with Databricks and EDP teams (reference: Swapnil's tenancy structure document), the following model has been established:

#### **Workspace Structure**
Each **workspace** is a physically separate Databricks environment with:
- Dedicated VPC and network isolation
- Independent security boundaries
- Separate AD group authentication
- Distinct compute resources

#### **Tenancy Within Workspace**
Within a single workspace, multiple **tenancies** (logical subdivisions) can coexist:
- Shared workspace infrastructure
- Logically separated storage folders
- Separate compute clusters per tenant
- Security group-based isolation

#### **BUK AIML Implementation**
For the AIML use case, we have **three separate workspaces** (NOT tenancies within one workspace):

| Workspace Name | VPC | Purpose | Rationale for Separation |
|----------------|-----|---------|--------------------------|
| **Prod Analytics** | VPC-Analytics | Data Science Lab | Development requires read/write access and experimentation freedom |
| **Prod Parallel** | VPC-Parallel | UAT/Integration Testing | Segregation of test from production for security and stability |
| **Prod** | VPC-Prod | Production Serving | Absolute lockdown - no interactive access permitted |

**Note:** Within the broader Barclays Databricks ecosystem, other teams (e.g., Fraud Analytics) have their own separate workspaces. The AIML team is **one tenant** within the overall Barclays Databricks tenancy model, but operates **three separate workspaces** for environment segregation.

---

## 3. Workspace Architecture

### 3.1 Three-Workspace Security Model

| Workspace | Equivalent Legacy Environment | Primary Persona | Access Model | Data Residency | Key Controls |
|-----------|------------------------------|-----------------|--------------|----------------|--------------|
| **Prod Analytics** | Build/Dev | Data Scientists, AIML Engineers | Read/Write | Temporary (perishable) | Unity Catalog access, 90-day retention, no data download |
| **Prod Parallel** | Test/UAT | AIML Team (validation), CI/CD Service Principals | Read-Only (human) / Automated (CI/CD) | Temporary (perishable) | Limited access - **under security review** |
| **Prod** | Production | RTB Team (via automation only) | No Interactive Access | Persistent (governed by EDP) | Service principals only, CR-gated deployments |

### 3.2 Workspace Segregation Rationale

#### **Why Not One Workspace with Multiple Tenancies?**

The security review confirmed that three separate workspaces are required because:

1. **Security Posture Differences:** Each environment has fundamentally different access requirements that cannot be adequately controlled within tenancy-level permissions
2. **Blast Radius Containment:** A security incident in Lab should not impact Production
3. **Regulatory Compliance:** Separation of duties and environment isolation for audit requirements
4. **Performance Isolation:** Training workloads (high CPU/GPU) should not impact production inference serving
5. **Network Security:** Different VPCs allow granular network controls and security group policies

#### **Why Lab (Prod Analytics) Exists**

Traditional banking might question: "Why allow access to production data in a Lab environment?"

**Answer:** Modern ML development requires:
- **Iterative experimentation** with production-representative data
- **Feature engineering** that must reflect actual data distributions
- **Model training** on historical production volumes (e.g., NBA 2.0 uses **49 interconnected models** trained on 3+ years of data)
- **Hyperparameter tuning** that requires multiple training runs

**Security Mitigations:**
- All data access via Unity Catalog (governed by EDP)
- No data download capability (prevents local copies)
- Perishable storage with automated retention (90-day maximum)
- Complete audit trail of data access
- VPC isolation from production environment

---

## 4. Detailed Component Explanation

### 4.1 Numbered Component Matrix

| # | Component Name | Type | Location | Purpose | Data Classification | Retention Policy | Key Security Controls |
|---|----------------|------|----------|---------|---------------------|------------------|----------------------|
| **1** | **Unity Catalog ↔ EDP Central Catalog Sync** | Data Interface | EDP + Databricks Control Plane | Bi-directional governed interface to bank's data marketplace | Inherits from source product | N/A (metadata only) | • Only approved Foundation/Consolidated/Business Data Products exposed<br>• Exceptions require CTO/CDAO approval<br>• Complete lineage tracking |
| **2** | **Databricks Bucket (Prod Analytics / Lab)** | S3 Storage | BUK AIML AWS Account (Federated) | Temporary storage for feature engineering outputs, training datasets, interim artifacts | Internal/Confidential (derived from production) | **Automated 90-day retention** (versioned deletion) | • "Data in flight" - perishable by design<br>• No permanent production data<br>• Versioned objects auto-deleted<br>• Encrypted at rest (AES-256)<br>• Access via IAM roles only |
| **3** | **EDP Datalake (Central Catalog)** | Data Marketplace | EDP Central AWS Accounts | Bank's authoritative data source and destination for CDPs | Varies (Public to Restricted) | Per EDP data product policy | • Source of all consumed data<br>• Destination for AIML CDPs<br>• Governed by EDP team |
| **4** | **Databricks Dedicated Cloud Infrastructure** | Control Plane | Databricks-managed + Barclays VPC | Orchestration layer that provisions and manages three workspaces | N/A (control plane) | N/A | • Manages workspace lifecycle<br>• Cluster orchestration<br>• Job scheduling |
| **5** | **Databricks Bucket (Prod Parallel)** | S3 Storage | BUK AIML AWS Account | Temporary storage for UAT/integration testing | Internal/Confidential | **Automated retention** (TBD: 30-90 days) | • Same controls as bucket #2<br>• Separate from Lab for audit clarity |
| **6** | **GitLab (Feature/Dev Branch)** | Source Control | Barclays GitLab | Code repository for Lab workspace notebooks and scripts | Internal | Per GitLab retention policy | • Standard branching model<br>• Code review required for merges<br>• Audit trail of all commits |
| **7** | **GitLab (UAT/Release Branch)** | Source Control | Barclays GitLab | Release branch for Prod Parallel deployment | Internal | Per GitLab retention policy | • Release tag required<br>• Approval gates for merges<br>• Triggers UAT CI/CD pipeline |
| **8** | **Nexus Model Registry** | Artifact Repository | Barclays Nexus | Central versioned repository for model binaries | Internal | Permanent (versioned) | • **Only model binaries - NO training data**<br>• Version control and rollback<br>• Immutable artifacts<br>• Checksum validation |
| **9** | **GitLab Runner (UAT)** | CI/CD Pipeline | Barclays CI/CD Infrastructure | Automates deployment to Prod Parallel workspace | N/A | N/A | • Service principal authentication<br>• Pulls code from GitLab (#7)<br>• Pulls model from Nexus (#8)<br>• May accept config parameters for throttling/canary |
| **10** | **GitLab Runner (Prod)** | CI/CD Pipeline | Barclays CI/CD Infrastructure | Production deployment pipeline | N/A | N/A | • **Triggered ONLY by approved CR**<br>• Executed by RTB team only<br>• Service principal authentication<br>• Full audit logging |
| **11** | **Model Execution & Monitoring (Prod)** | Compute Workload | Prod Workspace (VPC-Prod) | Production model serving and monitoring | N/A (processing only) | N/A | • **Zero interactive human access**<br>• Service principals only<br>• Monitoring and alerting<br>• Break-glass procedures documented |
| **12** | **Databricks Bucket (Prod)** | S3 Storage | BUK AIML AWS Account | Production bucket for final inference output | Internal/Confidential | **Permanent** (governed by EDP CDP policy) | • Output registered as official CDP<br>• Synced to EDP Central Catalog<br>• Encrypted at rest<br>• Immutable once published |
| **13** | **Tenant Account (e.g., Martech, MDP)** | Consumer Domain | Tenant AWS Accounts | Downstream consumer domains | Varies | Per tenant policy | • Access AIML CDP **exclusively via EDP Central Catalog**<br>• **Never direct access** to AIML buckets/accounts<br>• Governed by EDP access controls |

### 4.2 Component Deep-Dive

#### Component #1: Unity Catalog ↔ EDP Central Catalog Sync

**Critical Design Point:** This is the **ONLY** governed path for data to enter or exit the AIML environment. All data lineage flows through this interface.

**How It Works:**
1. EDP Data Platform team publishes approved data products to Central Catalog
2. Unity Catalog syncs metadata and access permissions
3. Data Scientists query via Unity Catalog in Databricks
4. Data remains in EDP buckets (not copied to AIML buckets initially)
5. AIML outputs (CDPs) are registered back to Central Catalog upon production deployment

**Exceptions:**
- **Raw bucket access:** In cases where data has not yet been promoted to a Foundation Data Product, AIML can request exception to access "base layer" or "prepared layer" buckets directly
- **Approval Required:** CTO and CDAO must approve exceptions
- **Current State:** Some exceptions in place for historical reasons (e.g., legacy SageMaker workflows)

**Data Product Types Consumed:**
- **Foundation Data Products (FDP):** Cleansed, standardized bank data
- **Consolidated Data Products (CDP):** Pre-aggregated or enriched datasets
- **Business Data Products (BDP):** Domain-specific business views

**Data Product Type Published:**
- **Consolidated Data Product (CDP):** AIML is classified as a CDP producer
  - Examples: Model inferences, feature stores, propensity scores
  - Published to Central Catalog for tenant consumption

#### Component #2: Databricks Bucket (Prod Analytics / Lab)

**Purpose:** Temporary workspace for data scientists during model development lifecycle.

**What Gets Stored Here:**
- Feature engineering intermediate outputs
- Training dataset snapshots (NOT raw production data copies)
- Model training artifacts (logs, checkpoints)
- Experiment results and metrics
- Temporary query results

**What Does NOT Get Stored Here:**
- Raw production data (accessed via Unity Catalog, not copied)
- Final model binaries (go to Nexus #8)
- Production inference outputs (go to bucket #12)

**Retention Policy Details:**
- **Current Implementation:** 90-day automated retention (aligns with existing analytics workspace precedent)
- **Under Discussion:** AIML team requested longer retention for complex use cases (e.g., NBA 2.0 with 49 models requiring 3+ years of training data iterations)
- **Security Requirement:** Retention period must be finite and automated - no manual intervention required
- **Versioning:** S3 versioning enabled; all versions deleted upon retention expiry
- **Principle:** Data is **perishable** - once model is promoted to production, lab artifacts are no longer needed

**Why This Bucket Exists:**
- Training ML models requires iterative experimentation
- Data Scientists need fast access to intermediate results
- Unity Catalog queries can be expensive for repeated access
- Performance optimization for model training workflows

**Why 90 Days is Being Questioned:**
- **NBA 2.0 Use Case:** Network of 49 interconnected models
  - Each model requires individual training and tuning
  - Iterative refinement across the network
  - Historical comparison of model versions
- **Training Data Volume:** 3+ years of historical data processed
- **Development Cycle:** From initial exploration to production-ready model can exceed 90 days

**Security Stance:**
- **Principle Agreed:** Data must be perishable with automated deletion
- **Timeline Under Discussion:** Balance operational needs vs. risk appetite
- **Compromise Being Explored:** Staged retention (e.g., 30 days for raw artifacts, 90 days for aggregated results)

#### Component #8: Nexus Model Registry

**Critical Security Control:** Nexus stores **ONLY model binaries** - absolutely NO training data, NO raw datasets, NO PII.

**What's Stored:**
- Serialized model files (pickle, ONNX, TensorFlow SavedModel, etc.)
- Model metadata (hyperparameters, training metrics, lineage)
- Model versioning information
- Checksums and signatures for integrity validation

**What's NOT Stored:**
- Training datasets
- Feature engineering pipelines (code is in GitLab)
- Inference results
- Any customer data or PII

**Security Measures:**
- Immutable artifacts (cannot modify published models)
- Version control with rollback capability
- Checksum validation on retrieval
- Access controlled via IAM roles
- Audit logging of all access

#### Component #11: Model Execution & Monitoring (Prod)

**Zero Interactive Access Policy:**

This is a **"break glass" environment** with the following constraints:

**Normal Operations:**
- All model execution via automated batch jobs
- Monitoring via centralized logging and dashboards
- Alerts routed to RTB team for response
- NO interactive logins permitted

**Emergency Access ("Break Glass"):**
- Documented procedure for emergency access
- Requires VP-level approval
- Time-limited access (e.g., 2 hours maximum)
- Full audit trail and mandatory post-incident review
- Used only for critical production incidents

**Monitoring & Observability:**
- Model performance metrics (accuracy, latency, throughput)
- Data drift detection
- Infrastructure health (CPU, memory, disk)
- Alerting thresholds and escalation procedures
- **Note:** Detailed monitoring architecture to be documented separately (mentioned as separate topic in security review)

---

## 5. Network Architecture & Security

### 5.1 VPC Design

Each workspace operates in a dedicated VPC to ensure network-level isolation:

| Workspace | VPC Name | CIDR Block | Subnets | Security Groups | Key Network Controls |
|-----------|----------|------------|---------|-----------------|---------------------|
| Prod Analytics | VPC-Analytics | TBD by EDP | Private subnets only | SG-Analytics-Databricks<br>SG-Analytics-Storage | • No direct internet access<br>• NAT Gateway for outbound (package downloads)<br>• VPC Endpoints for AWS services<br>• Direct Connect to on-prem |
| Prod Parallel | VPC-Parallel | TBD by EDP | Private subnets only | SG-Parallel-Databricks<br>SG-Parallel-Storage | • Same as Analytics<br>• Network ACLs prevent cross-VPC traffic |
| Prod | VPC-Prod | TBD by EDP | Private subnets only | SG-Prod-Databricks<br>SG-Prod-Storage<br>SG-Prod-Monitoring | • Most restrictive<br>• Egress limited to EDP endpoints<br>• No NAT Gateway (no package installs) |

### 5.2 Security Group Strategy

**Principle:** Security Groups are the primary enforcement mechanism for network isolation (as discussed in security review: "Everything is handled through security groups").

#### **Prod Analytics (Lab) Security Groups:**
```
SG-Analytics-Databricks:
  Inbound:
    - Source: Databricks Control Plane CIDR | Ports: 443, 6666-6690 (cluster comm)
    - Source: SG-Analytics-Databricks (self) | Ports: All (inter-cluster)
  Outbound:
    - Destination: SG-Analytics-Storage | Ports: 443 (S3 access)
    - Destination: Unity Catalog VPC Endpoint | Ports: 443
    - Destination: NAT Gateway | Ports: 80, 443 (package downloads)
    - Destination: Databricks Control Plane CIDR | Ports: 443

SG-Analytics-Storage:
  Inbound:
    - Source: SG-Analytics-Databricks | Ports: 443
  Outbound:
    - Destination: 0.0.0.0/0 | Ports: 443 (S3 via VPC Endpoint)
```

#### **Prod (Production) Security Groups:**
```
SG-Prod-Databricks:
  Inbound:
    - Source: Databricks Control Plane CIDR | Ports: 443, 6666-6690
    - Source: SG-Prod-Databricks (self) | Ports: All
  Outbound:
    - Destination: SG-Prod-Storage | Ports: 443
    - Destination: Unity Catalog VPC Endpoint | Ports: 443
    - Destination: Databricks Control Plane CIDR | Ports: 443
    - Destination: CloudWatch VPC Endpoint | Ports: 443 (monitoring)
    - NO NAT Gateway access (no package downloads in prod)
```

**Key Difference:** Production has NO outbound internet access via NAT Gateway, preventing any unauthorized egress.

### 5.3 Inter-VPC Communication

**Design Principle:** No direct communication between workspaces - all data sharing via EDP Central Catalog.

**Enforcement:**
- Network ACLs explicitly deny traffic between VPC CIDR blocks
- No VPC peering between AIML workspaces
- Transit Gateway rules (if applicable) deny AIML workspace interconnection

**Exception:** EDP Central Catalog is accessed via VPC Endpoints in each workspace - this is the controlled data sharing mechanism.

### 5.4 Connection to On-Premises & EDP

**Direct Connect Architecture:**
- Dedicated Direct Connect link from Barclays on-premises to AWS
- Private Virtual Interface (VIF) for secure, encrypted connectivity
- **No Internet Gateway** - all external connectivity via Direct Connect
- Access to EDP Datalake buckets via PrivateLink/VPC Endpoints

**Data in Transit Encryption (see Section 10.2):**
- Direct Connect provides physical isolation but data is not encrypted by default
- **Current Discussion:** Whether to add application-layer encryption (AES-256)
- **Performance Impact:** Testing shows ~14ms overhead for Base64 AES-256 encryption
- **Trade-off:** Security vs. performance for large data transfers
- **Status:** Under discussion with network security team (Richard's workstream)

### 5.5 Databricks CLI & Exfiltration Risk

**Critical Security Risk Identified:**

During the security review, a **HIGH PRIORITY** risk was identified:

**Risk:** Databricks allows users with appropriate workspace access to use CLI tools or IDE emulators (e.g., `databricks fs cp`, `%fs` magic commands) to download data from the Unity Catalog or workspace storage to their local laptops.

**Impact:**
- Data exfiltration outside controlled environment
- GDPR compliance violation (Right to be Forgotten cannot be guaranteed)
- Loss of data lineage tracking
- Potential data breach if laptop is compromised

**Current Mitigation Strategies Under Evaluation:**
1. **Databricks API-level controls** - Working with Databricks to disable download APIs
2. **Network egress filtering** - Block large data transfers via proxy inspection
3. **DLP (Data Loss Prevention)** - Endpoint DLP to detect/block suspicious downloads
4. **Audit & Alert** - Monitor for CLI usage patterns indicative of bulk downloads
5. **User behavior analytics** - Detect anomalous data access patterns

**Status:** Open workstream with Security + Databricks. Target resolution: December 2025.

**Temporary Mitigation:**
- Enhanced audit logging of all CLI commands
- User training on data handling policies
- Restricted workspace access to approved personnel only
- Quarterly access reviews

---

## 6. Data Flow Patterns

### 6.1 Primary Data Flow: Source to Consumption

```
[EDP Datalake (3)] 
    ↓ (Unity Catalog Sync)
[Unity Catalog (1)] 
    ↓ (Governed Query)
[Prod Analytics Workspace - Data Scientists] 
    ↓ (Feature Engineering)
[Lab Bucket (2) - Interim Storage - PERISHABLE]
    ↓ (Model Training)
[MLflow - Model Registry]
    ↓ (Model Publication)
[Nexus (8) - Model Binaries]
    ↓ (CI/CD UAT Deployment)
[Prod Parallel Workspace - Integration Testing]
    ↓ (CR Approval + CI/CD Prod Deployment)
[Prod Workspace (11) - Model Execution]
    ↓ (Inference Output)
[Prod Bucket (12)]
    ↓ (CDP Registration)
[EDP Central Catalog (3) - Published CDP]
    ↓ (Tenant Access)
[Tenant Account (13) - e.g., Martech MDP]
```

### 6.2 Data Flow Detailed Walkthrough

#### **Phase 1: Data Acquisition (Lab Environment)**

1. **Data Scientist** logs into **Prod Analytics Workspace** via SSO/AD authentication
2. **Query** Foundation or Consolidated Data Products via **Unity Catalog (1)**
3. Unity Catalog enforces access controls based on user role and data classification
4. **Data accessed in-place** from EDP Datalake (3) - NOT copied to AIML buckets initially
5. **Feature engineering** performed in Databricks notebooks/Spark jobs
6. **Intermediate results** written to **Lab Bucket (2)** for iterative development
7. **Training datasets** created from features (still in Lab Bucket 2)

**Key Principle:** Data is queried from EDP, transformed, and only intermediate results are stored temporarily in AIML buckets.

#### **Phase 2: Model Development (Lab Environment)**

8. **Model training** performed on Databricks clusters (compute-intensive)
9. **Hyperparameter tuning** and experimentation tracked via MLflow
10. **Training metrics** and checkpoints stored in Lab Bucket (2)
11. **Final model selection** based on performance criteria
12. **Model binary** exported and published to **Nexus Registry (8)**
13. **Code committed** to GitLab Feature Branch (6)

**Example Use Case: NBA 2.0**
- **Network of 49 models** (product propensity models)
- **Training data:** 3+ years of customer interaction history
- **Compute requirements:** High CPU/GPU for training (2x production serving capacity)
- **Development duration:** Can exceed 90 days due to model interdependencies
- **Lab Bucket usage:** Stores feature sets for all 49 models during development

#### **Phase 3: UAT Deployment (Prod Parallel Environment)**

14. **Code review** and **merge** to GitLab UAT/Release Branch (7)
15. **GitLab Runner (UAT) (9)** triggered automatically
16. **Pipeline actions:**
    - Pulls code from GitLab Release Branch (7)
    - Pulls model binary from Nexus (8)
    - Configures **Prod Parallel Workspace** compute resources (right-sized for testing)
    - Deploys notebooks and model to Parallel workspace
17. **Integration testing** performed:
    - End-to-end pipeline validation
    - Performance testing (latency, throughput)
    - Data quality validation
    - **Canary/Blue-Green testing** (if applicable - see Section 8.4)
18. **Test results** validated by AIML team
19. **Approval** obtained for production deployment

**Data Source for UAT:** Same Unity Catalog (1) access to production data - ensures testing with real data distributions.

#### **Phase 4: Production Deployment (Prod Environment)**

20. **Change Request (CR)** raised by AIML team
21. **CR approval** obtained (CAB process)
22. **RTB team** executes **GitLab Runner (Prod) (10)** - NO AIML team involvement
23. **Pipeline actions:**
    - Pulls code from approved release tag (GitLab 7)
    - Pulls model binary from Nexus (8)
    - Deploys to **Prod Workspace (11)** with production compute configuration
    - Configures monitoring and alerting
    - Validates deployment health checks
24. **Model execution (11)** begins processing production batch workloads
25. **Inference outputs** written to **Prod Bucket (12)**

**Compute Sizing:** Production inference typically requires ~50% of training compute (discussed in security review: "When we serve, we have strict terms EQ" - interpreted as more efficient/smaller compute for serving vs. training).

#### **Phase 5: Data Publication & Consumption**

26. **Inference output** in Prod Bucket (12) registered as **Consolidated Data Product (CDP)**
27. **CDP metadata** synced to **EDP Central Catalog (3)**
28. **Data governance** applied (access controls, classification, lineage)
29. **Tenant accounts (13)** discover CDP in Central Catalog
30. **Tenant queries** CDP via Unity Catalog in their own workspaces
31. **Data consumed** for downstream applications (e.g., Martech campaigns, dashboards)

**Example Consumer: Martech MDP (Marketing Data Platform)**
- **Use Case:** Customer propensity scores from NBA 2.0 models
- **Access Method:** Unity Catalog query from MDP workspace
- **Data Refresh:** Real-time or batch depending on SLA
- **No Direct Access:** MDP never accesses AIML buckets or workspaces directly

### 6.3 Exception Flow: Raw Bucket Access

**When Required:**
- Data has not yet been promoted to Foundation Data Product
- Urgent business need that cannot wait for FDP creation
- Data quality issues in existing FDP requiring base layer access

**Approval Process:**
1. AIML team submits exception request to EDP team
2. **CTO and CDAO approval required** (as stated in security review)
3. Temporary exception granted with time limit (e.g., 90 days)
4. Exception logged in risk register
5. Remediation plan required (promote to FDP or alternative solution)

**Security Controls for Exception:**
- Read-only access to base layer buckets
- IAM role with restricted permissions
- Audit logging enabled
- Quarterly review of exceptions to ensure they're temporary

**Current State:** Some exceptions in place from legacy SageMaker migration - these are being remediated as data products are formalized in EDP.

---

## 7. CI/CD Pipeline Flow

### 7.1 GitLab Branching Strategy

```
main (protected)
  ├── feature/* (Lab development)
  ├── release
