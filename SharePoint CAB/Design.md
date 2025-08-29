# I.T CAB System App – Prototype Build Task List

## Overview
Design and build a prototype I.T Change Advisory Board (CAB) system using:
- **SharePoint Online** (as the hosting platform).
- **GitHub** (as source control and CI/CD pipeline).
- **Microsoft 365 Message Center API** (to ingest Microsoft change updates).
- **Power Automate and/or SPFx** (for workflows and UI).
- **PowerShell / PnP / Graph API** (for deployment and data ingestion).

The prototype must be fully functional without placeholder/demo data.

---

## Task List

### 0.1 **Pre-requisites**
- [ ] Service Principals with required permissions
   - Deployment Service Principal
      - Used by GitHub Actions (CI/CD) for deploying SPFx solutions, running PnP.PowerShell scripts, and site/list provisioning.
      - Needs permissions for SharePoint site/app catalog and PnP.PowerShell certificate.
   - Ingestion/Automation Service Principal
      - Used for ongoing scheduled tasks (e.g., Microsoft Message Center ingestion via Graph API, Power Automate, or Azure Automation). 
      - Needs Microsoft Graph permissions (e.g., ServiceHealth.Read) and SharePoint list access.
- [ ] All secrets stored within Github Secrets to allow for their use in GitHub Actions workflows.
| Secret Name | Description |
|-------------|-------------|
| DEPLOYMENT_CLIENT_ID | Deployment service principal client ID |
| DEPLOYMENT_TENANT_ID | Azure AD tenant ID for deployment |
| DEPLOYMENT_CERTIFICATE | Auth for deployment service principal. This one has to be a certificate due to PnP restrictions |
| SHAREPOINT_APP_CATALOG_URL | App catalog site URL |
| SHAREPOINT_SITE_URL | Target SharePoint site URL |
| INGESTION_CLIENT_ID | Ingestion/automation service principal ID |
| INGESTION_CLIENT_SECRET | Auth for ingestion/automation principal |
| WEBHOOK_URLS | Notification endpoints |
- [ ] PnP PowerShell certificates generated

### 1. **Repository Setup**
- [ ] Create a **GitHub Repository** for the project.
- [ ] Structure folders for:
/
├── .github/
│   └── workflows/                # GitHub Actions workflows (CI/CD)
├── deployment/
│   └── scripts/                  # PowerShell scripts for provisioning, ingestion, etc.
├── src/
│   └── spfx-solution/            # SPFx solution (React web parts)
└── README.md  

---

### 2. **SharePoint Site Preparation** (PnP PowerShell deployment)
- [ ] Create a **SharePoint Communication Site**
- [ ] Configure site permissions:
  - CAB Admins, Approvers, Reviewers, and Users groups.
- [ ] Create SharePoint **Lists**:
  - `Internal Change Requests` (Title, Description, Status, Priority, Created Date, Modified Date).
  - `Microsoft Changes` (Title, Description, Impact, Category, Status, Published Date, Ingested Date, **User Priority**).
    - Users can grade or set priority for each Microsoft change (e.g., High, Medium, Low, or numeric score).
    - The SPFx dashboard should allow filtering so users only see changes matching their selected priority or categories (e.g., SharePoint, Power Automate).
    - Optionally, use the **Category** field to further filter changes by product/service.
  - `Change Log` (for audit history).

---

### 3. **SPFx Front-End Development**
#### Summary:
- You’re building React-based web parts with SPFx, packaging them, and deploying to SharePoint so users can view, submit, and manage change requests through a modern UI.
- [ ] Initialize a **SharePoint Framework (SPFx)** solution:
  - Use React as the UI framework.
- [ ] Develop web parts:
  - **Change Dashboard**: List all internal and Microsoft changes with filters, including user priority and category filtering for Microsoft changes.
  - **Change Submission Form**: Form for submitting internal change requests.
  - **Change Details Page**: View and manage specific change requests.
- [ ] Package and deploy to **SharePoint App Catalog** using:
  - `gulp bundle --ship`
  - `gulp package-solution --ship`
- [ ] Deploy web parts to target SharePoint site.

---

### 4. **Microsoft Message Center Ingestion Process**

#### 4.1. **Microsoft 365 API Setup**
- [ ] Register an **Azure AD App Registration**:
  - Grant required Microsoft Graph permissions (`ServiceHealth.Read`).
- [ ] Generate **client secret** for the app.

#### 4.2. **Ingestion Script**
- [ ] Write a **PowerShell Script** using **Graph API**:
  - Authenticate using Azure AD App.
  - Call Microsoft 365 Message Center endpoint.
  - Parse updates.
  - Push updates to the `Microsoft Changes` SharePoint list using: **PnP.PowerShell**

- [ ] Ensure:
  - No placeholder content.
  - Real updates from Microsoft Message Center are ingested.

#### 4.3. **Scheduling**
- [ ] Deploy script to **GitHub Actions** for:
  - daily ingestion.
- [ ] Log ingestion runs and errors to a SharePoint `Change Log`.

---

### 5. **Change Approval Workflow**
- [ ] Build workflows using **Power Automate**:
  - **Internal Change Requests Workflow**:
    - On new item submission.
    - Notify reviewers.
    - Collect approvals.
    - Update request status (`Approved`, `Rejected`, `In Progress`, `Completed`).
  - **Microsoft Changes Workflow**:
    - Auto-tagged as `Microsoft`.
    - Processed similarly to internal requests.

---

### 6. **CI/CD Pipeline Setup**
- [ ] Write **GitHub Actions** workflows for:
  - On push to `main`:
    - Build SPFx solution.
    - Package solution.
    - Deploy package to SharePoint App Catalog using:
      - **PnP.PowerShell**, or
      - Custom deployment scripts via Azure CLI.
- [ ] Automate deployment to SharePoint site after successful build.

---

### 7. **Power BI Dashboard (Optional for Prototype)**
- [ ] Create a **Power BI report** connected to:
  - `Internal Change Requests` list.
  - `Microsoft Changes` list.
- [ ] Publish dashboard to SharePoint site via Power BI Web Part.

---

### 8. **Testing and QA**
- [ ] Perform end-to-end ingestion tests:
  - Verify real Microsoft Message Center updates are pulled in.
- [ ] Submit internal change requests via UI.
- [ ] Review and approve requests via Power Automate workflow.
- [ ] Validate the change dashboard displays combined data.

---

### 9. **Documentation**
- [ ] Document:
  - System architecture.
  - Deployment process.
  - Data ingestion script setup.
  - User instructions for submission and approval processes.
  - GitHub Actions workflows.

---

### 10. **Final Validation**
- [ ] Ensure no placeholders or demo data.
- [ ] Ingested updates should reflect actual Microsoft change notifications.
- [ ] Functional CAB system operating in SharePoint with GitHub-driven deployments.

---
