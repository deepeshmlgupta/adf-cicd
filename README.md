# ADF CI/CD — GitHub Actions

This repository contains the CI/CD pipeline for **Azure Data Factory (ADF)** workloads, built on GitHub Actions. The goal is to eliminate manual deployments — any change a data engineer makes in ADF Studio is automatically validated and promoted through environments via Pull Requests, with human approval gates before anything reaches UAT or Production.

> **ADF Dev Workspace** is the single source of truth. It has Git integration enabled, which means every change saved in ADF Studio is automatically committed to this repository. UAT and Prod never get touched manually — they receive changes only through this pipeline.

---

## Architecture

The diagram below shows the full end-to-end flow — from a data engineer making a change in ADF Studio, all the way through to production deployment.

![ADF CI/CD Architecture](https://github.com/user-attachments/assets/999ea283-62be-4197-960e-77a80263ee08)
---

## Branch Strategy

Changes always flow in one direction: `feature` → `dev` → `uat` → `main`. There are no shortcuts — every promotion goes through a Pull Request, and merging into UAT or Prod requires an approver to sign off before the deployment runs.

| Branch | Environment | Deployment Behaviour |
|---|---|---|
| `feature/*` | Developer working branch | No deployment — working area only |
| `dev` | Dev ADF Workspace | Auto-deploys on merge — no approval needed |
| `uat` | UAT ADF Workspace | Pauses for approver sign-off before deploying |
| `main` | Prod ADF Workspace | Pauses for approver sign-off before deploying |

**Direct commits to `dev`, `uat`, or `main` are not allowed.** All changes must come in through a Pull Request so there is always a review trail.

---

## Pipelines

### CI — Validation Pipeline

This pipeline runs automatically on **every Pull Request**. Before anyone can merge their changes, this pipeline checks that all ADF JSON files are valid — correct syntax, no broken pipeline references, no misconfigured triggers. If the check fails, the merge button is blocked until the issues are fixed.

No Azure credentials are needed here. All validation happens locally on the GitHub runner by reading the files already in the repository.

### Deploy — Build & Deploy Pipeline

This pipeline runs automatically when a PR is **merged** into `dev`, `uat`, or `main`. It handles both building the deployment package and pushing it to the right ADF environment.

```
Step 1 — Resolve which environment to deploy to based on the branch
Step 2 — Validate the merged code + generate ARM templates + save as versioned artifact
Step 3 — (UAT and Prod only) Pause and wait for approver sign-off
Step 4 — Stop any currently running ADF triggers
Step 5 — Deploy ARM template to the target ADF workspace
Step 6 — Restore only the triggers that were running before deployment
Step 7 — Write a deployment summary to the pipeline log
```

> **On triggers:** The pipeline saves which triggers were running *before* deployment and restores only those afterwards. This prevents accidentally restarting triggers that were intentionally stopped before the deployment window.

---

## Prerequisites

Before the pipelines can run, the following needs to be in place.

### Azure

- **Dev ADF** must have Git integration enabled and connected to this repository
- **UAT and Prod ADF** should be in live mode — no Git integration needed on these
- A **Service Principal** is needed per environment. It requires the **Data Factory Contributor** role assigned directly on the target ADF resource (not at subscription or resource group level)

### GitHub — Environments

Create three environments under **Repository Settings → Environments**: `adf-dev`, `adf-uat`, `adf-prod`

- `adf-dev` — no approval gate, deploys automatically
- `adf-uat` and `adf-prod` — enable **Required reviewers** and add the appropriate approvers

Each environment must have the following configured:

| Name | Type | Description |
|---|---|---|
| `ARM_CLIENT_ID` | Secret | Service Principal Application (Client) ID |
| `ARM_CLIENT_SECRET` | Secret | Service Principal Client Secret |
| `ARM_TENANT_ID` | Secret | Azure Active Directory Tenant ID |
| `TARGET_ADF_RESOURCE_ID` | Variable | Full Azure Resource ID of the target ADF instance |

The `TARGET_ADF_RESOURCE_ID` can be copied from **Azure Portal → ADF resource → Properties → Resource ID**. It follows this format:
```
/subscriptions/{sub-id}/resourceGroups/{rg-name}/providers/Microsoft.DataFactory/factories/{adf-name}
```

The pipeline automatically extracts the subscription, resource group, and factory name from this single value — no need to store them separately.

### GitHub — Repository Variable

One variable is needed at the repository level (not inside an environment). This is used by the CI validation pipeline, which runs on Pull Requests before any environment context is available.

| Name | Value |
|---|---|
| `CI_FACTORY_ID` | Resource ID of the Dev ADF — same value as `adf-dev`'s `TARGET_ADF_RESOURCE_ID` |

Go to **Repository Settings → Secrets and variables → Actions → Variables** to add this.

### GitHub — Branch Protection Rules

Branch protection ensures the CI check must pass before any PR can be merged. Configure this for `dev`, `uat`, and `main` under **Repository Settings → Branches → Add rule**:

- ✅ Require a pull request before merging
- ✅ Require status checks to pass → select the CI validation check

---

## How to Deploy a Change

This is the standard day-to-day flow for a data engineer making a change.

1. Open the **Dev ADF Workspace** in ADF Studio and make your changes
2. In ADF Studio, save your work to a `feature/your-change-name` branch
3. Go to GitHub and raise a **PR from `feature/*` → `dev`**
   - CI validation runs automatically — fix any errors if it fails
   - Once it passes, merge the PR → pipeline auto-deploys to Dev ADF
4. When the change is ready for UAT, raise a **PR from `dev` → `uat`**
   - CI validation runs again on the merge candidate
   - Once merged, the pipeline pauses and notifies the UAT approver
   - Approver reviews and approves → pipeline deploys to UAT ADF
5. After UAT sign-off, raise a **PR from `uat` → `main`**
   - Same process — CI check, merge, approver notified, approve → deploys to Prod ADF
