# PAPU [![](https://img.shields.io/github/license/sourcerer-io/hall-of-fame.svg?colorB=ff0000)](https://github.com/rsanimesh/PAPU/blob/master/LICENSE)  [![](https://img.shields.io/badge/Rahul-Sinha-brightgreen.svg?colorB=ff0000)](http://sinharahul.com/)

Personal Artificial Programmable Unit.

Understood. I will revise the email draft and suggest the Confluence update, incorporating two key changes:
 * Question 3 (CSM Usage): Introduce ambiguity regarding which CSM (Dev or Prod) is used for the Prod Analytics workspace, while still affirming Prod CSM for the latter two.
 * GitLab/Databricks Connection: Integrate Santanu's point about using secrets for the GitLab-to-Prod Analytics connection, emphasizing the security of this method.
📧 Revised Draft Email Response
Subject: RE: BUK AI ML Databricks Architecture Call - Follow-up on Controls, Segregation, and Connectivity
Hi Swapnil, Rahul, Cameron, and Goutam,
Thank you again for the walkthrough and your valuable feedback. We have recorded the details and will update the architecture ticket accordingly, including a few clarifications on connectivity and resource management.
1. Controls on Temporary S3 Bucket (No. 3)
The bucket designated for temporary data storage (e.g., feature engineering outputs, interim artifacts) is secured with the following controls:
 * Access Restriction: Access to this bucket is strictly restricted to the Service Principal associated with the Prod Analytics Workspace (Development/Lab) only.
 * No Downstream Access: As confirmed, there is absolutely no downstream access provisioned for this temporary storage bucket to ensure data isolation.
 * Other Controls: The bucket will adhere to standard security policies, including encryption at rest and appropriate lifecycle policies (e.g., retention policy) to minimize the storage window for transient data.
2. Tenancy Segregation
Regarding segregation within the Production Analytical environment:
 * The current design relies on physical workspace separation (Prod Analytics for Dev, Prod Parallel for UAT, Prod for Inference) and Role-Based Access Control (RBAC) to provide segregation by persona and environment lifecycle stage.
 * We confirm that the current architecture does not implement further tenancy segregation based on individual models or model purpose within the workspaces. All models within a given workspace will share the same underlying compute/storage plane, secured by Unity Catalog governance.
3. GitLab Connection to Prod Analytics
To ensure secure establishment of the connection between GitLab and the Prod Analytics workspace:
 * We will leverage the platform's secret management capability.
 * This approach is considered legitimate as it ensures no one has direct access to the credentials/password, and we avoid saving the password directly in GitLab environment variables.
4. CSM Usage and Guild Engagement
We have reviewed the resource tracking requirements and confirmed the allocation mechanism:
 * The Prod Parallel and Prod workspaces will be provisioned under the Production CSM.
 * The Prod Analytics workspace, which serves as the Development/Data Science Lab environment, will align its CSM allocation based on the final financial mapping of the underlying development environment resources.
 * We will ensure alignment with the relevant financial and technical guilds/oversight groups to confirm the tagging and resource allocation mechanism meets all required governance standards.
Please let us know if you have any follow-up questions on these points so we can update the ticket promptly.
Thanks and Regards,
[Your Name/Team Name]
📝 Suggested Confluence Update
You should update the Section 2. Architecture Overview or add a new bullet point under the CI/CD Pipeline (Path to Production) section to incorporate the security and CSM points:
Under Section 2. Architecture Overview (or a dedicated section on Control Plane)
> Resource Allocation and Tracking
> The Prod Parallel and Prod workspaces are provisioned under the Production CSM for resource tracking. The Prod Analytics workspace (Development/Lab) is provisioned under the appropriate CSM based on development environment resource classification.
> 
Under Section 4.2. CI/CD Pipeline (Path to Production)
> Step 1 (Development): Code is written in the Lab, pushed to the GitLab Dev Branch. The final model binary is published to Nexus.
>  * Secure Connectivity Note: The connection between GitLab and the Prod Analytics workspace is established using secret management capabilities to ensure credentials are not exposed to users or stored as plaintext environment variables. This maintains a secure, non-interactive connection for automated CI/CD processes.
> 
Which of these documents (email or Confluence update) would you like to finalize first?
