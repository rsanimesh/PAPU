Here is the draft for your **`README.md`** file. You can save this as a text file (`.txt`), a Word doc, or a Markdown file (`.md`) at the very top of your project folder.

This document acts as the "map" for your team, ensuring the new structure stays clean.

---

# Project NBA 2.0 Migration - Structure & Index

**Last Updated:** January 2026
**Project Status:** Execution / Migration

## 1. Folder Structure Overview

This repository is organized by lifecycle phase. Please save files in their respective folders, not in the root directory.

* **📂 00_Project_Management**
* **Use for:** Project timelines, Gantt charts, dependency trackers, and resource planning.
* **Key File:** `NBA2.0_PM_MasterPlan_&_Roles_v1.0.xlsx` (Contains the detailed plan, dependencies, and User Access Role definitions).


* **📂 01_Architecture_&_Design**
* **Use for:** High-Level Design (HLD) documents, solution architecture diagrams, and Confluence reference links.
* **Key File:** `NBA2.0_Arch_MLOps_Diagram_v0.1.jpg` (Visual flow of Nexus/Gitlab/Databricks integration).


* **📂 02_Governance_&_Security**
* **Use for:** Compliance trackers, CSO approvals, PII confirmation logs, and Database access requests.
* **Key File:** `NBA2.0_Gov_CSO_Approval_Tracker_v1.0.xlsx` (Tracks CSO/DB Owner approvals per database).


* **📂 03_Data_Engineering**
* **Use for:** Feature store definitions, data lineage, path mappings to EDP, and data dictionaries.
* **Key File:** `NBA2.0_Data_FeatureStore_MASTER_v1.0.xlsx` (The single source of truth for BUK/Non-BUK features and descriptions).


* **📂 04_MLOps_&_Deployment**
* **Use for:** CI/CD pipeline configurations, Environment setup instructions (Prod Analytics/Parallel/Prod), and Job definitions.
* **Key File:** MLOps pipeline documentation and environment naming conventions.


* **📂 99_Archive_&_Drafts**
* **Use for:** Old file versions, scratchpads, and superseded exports.
* *Note:* If you are unsure if a file is current, look here first.



---

## 2. Naming Convention Standard

To maintain order, all new files must follow this naming convention:
`NBA2.0_[Category]_[ContentName]_[Version]`

**Examples:**

* `NBA2.0_PM_StatusReport_v1.docx`
* `NBA2.0_Data_IngestionSpec_v2.xlsx`
* `NBA2.0_Arch_Diagram_v1.png`

---

## 3. "Where do I find...?" (Quick Lookup)

| If you are looking for... | Open this file | Located in Folder |
| --- | --- | --- |
| **Project Timelines** | `NBA2.0_PM_MasterPlan_&_Roles_v1.0.xlsx` | 📂 00_Project_Management |
| **User Roles & Access** | `NBA2.0_PM_MasterPlan_&_Roles_v1.0.xlsx` (See 'Roles' tab) | 📂 00_Project_Management |
| **Feature Definitions** | `NBA2.0_Data_FeatureStore_MASTER_v1.0.xlsx` | 📂 03_Data_Engineering |
| **CSO / PII Approvals** | `NBA2.0_Gov_CSO_Approval_Tracker_v1.0.xlsx` | 📂 02_Governance_&_Security |
| **Confluence Links** | `NBA2.0_Ref_Confluence_Links_v1.0.docx` | 📂 01_Architecture_&_Design |
| **MLOps Diagrams** | `NBA2.0_Arch_MLOps_Diagram_v0.1.jpg` | 📂 01_Architecture_&_Design |

---

## 4. Important Notes regarding "Features" Files

* **DO NOT USE:** `NBA2.0 features 1.xlsx` or `NBA Features Summary_OLD.xlsx`. These are obsolete drafts.
* **USE ONLY:** `NBA2.0_Data_FeatureStore_MASTER_v1.0.xlsx`. This file contains the validated "Description", "IsBSL" flags, and correct data sources.

---

**Next Steps for the User:**

1. Create the folders listed above.
2. Rename your files according to the table in the previous chat.
3. Save this text as `README.md` or `README.txt` in the main folder.
