ko# PAPU [![](https://img.shields.io/github/license/sourcerer-io/hall-of-fame.svg?colorB=ff0000)](https://github.com/rsanimesh/PAPU/blob/master/LICENSE)  [![](https://img.shields.io/badge/Rahul-Sinha-brightgreen.svg?colorB=ff0000)](http://sinharahul.com/)

Personal Artificial Programmable Unit.


 Here is a polished and professional version of the email based on the notes in your image.
Subject: Inquiry regarding NBA2.0 Model Migration and Feature Sourcing
Hi Greta,
I hope this email finds you well.
I am reaching out with a few clarifying questions regarding the ongoing migration of the NBA2.0 models. As we understand, the current models in the prep environment consume features from both the Feature Store and the BSL layer, and the goal of the migration is to transition to sourcing all features from a single source: the Feature Store.
To help us better understand the roadmap, could you please clarify your approach regarding the following points?
 * On-Premise vs. Databricks Sourcing: Are you planning to update the models running on-premise to consume inputs exclusively from the Feature Store (phasing out the BSL layer)? Or, will this transition to a single source only happen directly on Databricks, while the on-premise systems continue taking inputs from both the Feature Store and BSL?
 * Validation and Drift Verification: If you are opting for the second approach (keeping the dual-source on-premise while updating Databricks), what steps will you be taking to compare the models across these two environments? Specifically, we would like to understand the methodology for verifying that moving features from BSL to the Feature Store does not introduce any discrepancies or feature drift.
Thank you in advance for your time and clarification. Let me know if it would be easier to discuss this over a quick call.
Best regards,
[Your Name]
Would you like me to adjust the tone to be slightly more formal, or is this ready for you to send?
