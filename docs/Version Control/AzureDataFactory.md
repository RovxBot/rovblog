# Design Document: Client Data Ingestion & Reporting Platform

## Objective

To implement a scalable, secure, and automated data pipeline that ingests monthly data from client APIs, stores it in Microsoft Fabric OneLake, and generates customized Power BI reports through a controlled Dev/Test/Prod deployment process. Each client’s data and reports must be securely separated, in accordance with ISO certification requirements.



## 1. System Overview

Core Technologies
* Ingestion: Azure Data Factory (ADF)
* Storage: Microsoft Fabric OneLake (Delta Lake format)
* Processing: Fabric Lakehouse (with optional Dataflows Gen2)
* Reporting: Power BI with Deployment Pipelines
* Version Control: GitHub
* Security: Azure RBAC, Row-Level Security (RLS), OneLake folder-level isolation



## 2. Architecture Summary

**High-Level Flow**
1.	Monthly Trigger in ADF initiates data collection for each client.
2.	Data Ingestion from external client APIs using parameterized pipelines.
3.	Data Storage in OneLake with client-specific folder paths.
4.	Power BI Workspaces per client access only their data.
5.	Power BI Deployment Pipelines manage Dev → Test → Prod progression.
6.	Reports are securely shared with individual clients.



## 3. Data Ingestion Layer (Azure Data Factory)

**Pipeline Design**
* Parameterization: Accepts client_id, api_url, token.
* Scheduling: Monthly triggers per client.
* Secrets: Stored in Azure Key Vault.

**Steps**
1.	Connect to client API (REST connector)
2.	Fetch and parse data (JSON/CSV/XML)
3.	Write data to OneLake as Delta files:

```
/Clients/{client_id}/yyyy-MM/data.delta
```



## 4. Data Storage Layer (Microsoft Fabric OneLake)

**Storage Model**
* Shared Lakehouse for all clients
* Client-specific folders to enforce isolation
* Data stored in Delta format for optimal querying

**Example Structure**

```
OneLake Workspace: Client-Data-Lakehouse
/
├── Clients/
│   ├── ClientA/
│   │   └── 2025-07/
│   │       └── data.delta
│   └── ClientB/
│       └── 2025-07/
│           └── data.delta
```


## 5. Reporting Layer (Power BI)

**Workspace Strategy**

Each client receives their own Power BI workspaces:

- ClientA-Dev
- ClientA-Test
- ClientA-Prod
- ClientB-Dev
- ClientB-Test
- ClientB-Prod

**Deployment Pipeline**
* Dev: Development and initial connection to client data
* Test: Internal QA and refresh validation
* Prod: Final client-ready reports

**GitHub Integration**
* PBIX files stored and versioned in client-specific GitHub repos
* Power BI REST API or Fabric GitHub Actions used for CI/CD


## 6. Data Access & Security

**Access Control**
* OneLake Folder Isolation: Each Power BI dataset only connects to the client’s folder.
* RBAC: Azure AD roles restrict access to each workspace.
* Optional RLS: If multi-client model is needed (not recommended here)

**Sharing**
* Power BI Apps: Used to bundle and share client reports
* Power BI Embedded: Optional for embedding into client portals with token-based access

**Compliance**
* All pipelines and storage follow ISO-compliant best practices
* Audit logs and access control are enforced via Azure


## 7. Automation & Monitoring

**Automation**
* ADF pipeline triggers: Monthly per client
* Dataset refreshes: Scheduled or event-triggered post-ingestion
* GitHub Actions deploy ADF and Power BI artifacts from repo

**onitoring Tools**
* ADF Activity Monitoring
* Power BI Refresh History
* Azure Monitor for alerts and logs


## 8. Naming Conventions

**Workspaces**

{ClientName}-{Stage}
Examples:
- AcmeCorp-Dev
- AcmeCorp-Test
- AcmeCorp-Prod

**OneLake Paths**
```
/Clients/{client_id}/{yyyy-MM}/data.delta
```


## 9. Demo Setup Guide (Full Walkthrough)

**Step 1: Provision Infrastructure**
* Create Resource Group
* Create OneLake Lakehouse workspace
* Create Key Vault and set API secrets

**Step 2: Create and Publish GitHub Repo**
* Use a repo template with ADF pipeline JSON, ARM/Bicep deployment scripts
* Push to GitHub and configure Actions

**Step 3: Deploy Azure Data Factory**
* Use ARM/Bicep to deploy ADF instance
* Deploy ingestion pipeline using GitHub Actions on push to main or publish branch

**Step 4: Configure ADF Pipeline**
* Parameterize pipeline for client_id, api_url, and auth_key
* Set monthly trigger
* Store secrets in Key Vault, reference them in Linked Services

Step 5: Setup OneLake Folder
* Create /Clients/DemoClient/yyyy-MM/ folder
* Confirm pipeline writes data to Delta file format

**Step 6: Create Power BI Workspaces**
* Create Dev, Test, Prod workspaces using Power BI REST API or manually
* Register them in Fabric deployment pipeline UI

**Step 7: Version Control Power BI Reports**
* Save reports as PBIP format in GitHub
* On PR or merge to main, deploy report via GitHub Action or manual trigger

**Step 8: Connect Power BI Dataset to OneLake**
* Use delta/Parquet connector to point to the demo folder
* Load and model dataset

**Step 9: Secure Access**
* Use Azure AD security groups to restrict workspace access
* Create Power BI App and share with demo users


## 10. Future Enhancements
* Implement Azure Purview for data catalog and classification
* Automate workspace and pipeline creation using PowerShell + REST APIs
* Add retry logic and alerting to ADF pipelines
* Expand GitHub workflows to support multiple branches and environments


## 11. Conclusion

This design ensures scalable, secure, and maintainable data processing across multiple clients. It enables customized reporting through isolated workspaces and centralized storage, while enforcing strict data segregation and compliance with ISO standards. The added step-by-step demo guide supports rapid prototyping and testing of the architecture in your own environment.