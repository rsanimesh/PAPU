# PAPU [![](https://img.shields.io/github/license/sourcerer-io/hall-of-fame.svg?colorB=ff0000)](https://github.com/rsanimesh/PAPU/blob/master/LICENSE)  [![](https://img.shields.io/badge/Rahul-Sinha-brightgreen.svg?colorB=ff0000)](http://sinharahul.com/)

Personal Artificial Programmable Unit.


Here's a professional, concise, and actionable meeting agenda tailored to your goal: brainstorming dependencies, feasibility, and especially the hidden/potential challenges of deploying NBA 2.0 (Next Best Action 2.0) in Databricks.

**Meeting Title:** NBA 2.0 on Databricks – Feasibility, Dependencies & Hidden Challenges Brainstorm  
**Objective:** Assess technical and operational feasibility of migrating/deploying NBA 2.0 to Databricks, identify all dependencies, and surface hidden risks and challenges  
**Date & Time:** [Insert date/time]  
**Duration:** 60–75 minutes  
**Attendees:** [You], Data Engineering Lead, ML Engineering Lead, Databricks Admin/Architect, NBA 2.0 Product Owner, Security/Compliance Lead, (optional) Cost/FinOps representative  
**Platform:** Teams/Zoom + Miro/Jamboard or Databricks Notebooks for live collaboration  

### Agenda

| Time     | Item                                                                 | Owner         | Objective / Output                                      |
|----------|----------------------------------------------------------------------|---------------|---------------------------------------------------------|
| 0:00–0:05 | Welcome, meeting objective & expected outcomes                       | You           | Align everyone on why we are here                       |
| 0:05–0:10 | Quick recap of NBA 2.0 current architecture & key components        | You / Product | Level-set: batch vs real-time, models, data volume, latency requirements |
| 0:10–0:25 | Feasibility check – Can Databricks technically support NBA 2.0?     | Databricks Architect | Discuss Unity Catalog, Delta Lake, MLflow, DBSQL, Workflows, Serverless, Model Serving, Jobs Compute vs All-Purpose vs Serverless |
|          |                                                                      |               | • Real-time/scoring latency requirements<br>• Feature store needs (online/offline)<br>• Multi-region / multi-cloud constraints |
| 0:25–0:45 | Dependencies mapping (must-haves before we can even start)           | All           | Whiteboard/Miro session                                |
|          | • Data ingestion pipelines (Kafka, Event Hubs, etc.)<br>• Feature pipeline (currently Spark/Flink?)<br>• Model training & registry (MLflow?)<br>• Serving (currently KServe/Triton/Seldon? → Databricks Model Serving)<br>• Authentication/Authorization (OKTA/SCIM, Table ACLs, Unity Catalog)<br>• Secrets management<br>• CI/CD (GitHub Actions → Databricks Repos + Asset Bundles)<br>• Monitoring (Ganglia → Databricks Lakeview dashboards, custom metrics to Prometheus/Grafana) |               | Clear dependency tree + owners                          |
| 0:45–1:05 | Hidden & common Databricks challenges that bite NBA-type workloads  | All (round-table) | Explicitly call out the “gotchas” most teams discover too late |
|          | Expected pain points to discuss:<br>• Cluster start-up time & cold-start latency for real-time scoring<br>• Cost unpredictability (Jobs Compute vs Serverless vs Pro/SQL warehouses)<br>• Unity Catalog governance at scale (thousands of tables, dynamic masking)<br>• Delta Live Tables vs traditional Jobs for feature pipelines (reliability, backfills)<br>• Model Serving concurrency limits and autoscaling behaviour<br>• Photon acceleration limitations with certain UDFs/ML inference<br>• IP whitelisting / PrivateLink / VPC peering complexities<br>• Audit log volume and cost<br>• Workspace clutter and asset lifecycle management<br>• Handling PII and GDPR “right to be forgotten” in immutable Delta tables<br>• Cross-workspace or cross-region dependencies |               | List of “hidden challenges” with severity & mitigation ideas |
| 1:05–1:15 | Effort & cost rough order of magnitude (ROM)                         | Data Eng + Databricks Architect | Very high-level LoE and monthly run cost estimate       |
| 1:15–1:20 | Risks, blockers & open questions                                     | You            | Capture in parking lot                                  |
| 1:20–1:25 | Next steps & owners                                                  | You            | • PoC scope (if we decide to go ahead)<br>• Deep-dive sessions needed<br>• Decision date                            |

**Pre-read (send 24h before):**  
- One-pager with current NBA 2.0 architecture diagram  
- Current monthly compute cost and SLAs (latency, throughput)  
- Databricks workspace inventory (clusters, jobs, warehouses in use today)

**Desired Outcome:**  
Clear go / no-go signals, full dependency map, prioritized list of hidden challenges with mitigation ideas, and concrete next steps (PoC or alternative paths).

Let me know if you want me to turn this into a Google Doc/Word template or add more Databricks-specific gotchas relevant to your stack!