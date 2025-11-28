# PAPU [![](https://img.shields.io/github/license/sourcerer-io/hall-of-fame.svg?colorB=ff0000)](https://github.com/rsanimesh/PAPU/blob/master/LICENSE)  [![](https://img.shields.io/badge/Rahul-Sinha-brightgreen.svg?colorB=ff0000)](http://sinharahul.com/)

Personal Artificial Programmable Unit.

Subject: Feature Store Architecture: Temporary vs. Strategic Approach

Currently, our AWS Feature Store operates on a temporary architecture where data is synced from the on-premise BIW2 data lake. This data is subsequently consumed within the Databricks workspace by data scientists and jobs via Unity Catalog. However, this route creates a dependency on our on-premise systems.

Our strategic roadmap is to build a Feature Store sourced directly from EDP data in AWS. In this model, Databricks ETL pipelines will generate features and sync them back to the AWS Feature Store (S3). These features will then be available for consumption within Databricks via Unity Catalog, effectively removing the on-premise dependency.
