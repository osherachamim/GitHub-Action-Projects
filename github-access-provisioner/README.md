# GitHub Access Provisioner

Automated end-to-end pipeline for provisioning GitHub repository access via Azure AD and GitHub Enterprise Managed Users (EMU).

When triggered, the pipeline:
1. Assigns an Azure AD group to the GitHub Enterprise Application
2. Triggers on-demand SCIM provisioning to sync the group to GitHub immediately
3. Creates a GitHub Team linked to the IDP external group
4. Assigns **Write** permission to the team on its corresponding repository

---

## Architecture

```
Freshdesk Service Catalog
  │  User submits request with repoName
  │
  ▼
Freshdesk Automation (Webhook)
  │  POST /repos/IT/github-access-provisioner/dispatches
  │  { "event_type": "provision-github-access", "client_payload": { "repoName": "myrepo" } }
  │
  ▼
GitHub Actions (repository_dispatch)
  │  Constructs group name: AG-GitHub-W-<repoName>
  │
  ▼
provision-groups-ondemand.py
  │
  ├── Phase 1 — Azure AD
  │     ├── Assign group to Enterprise App (idempotent)
  │     └── Trigger on-demand SCIM provisioning
  │
  ├── Phase 2 — GitHub Team
  │     ├── Create team (idempotent)
  │     └── Link team to IDP external group
  │
  └── Phase 3 — Repository Permission
        └── Assign Write permission to team on repo (idempotent)
```

---

## Naming Convention

| Component | Pattern | Example |
|-----------|---------|---------|
| Azure AD Group | `AG-GitHub-W-<reponame>` | `AG-GitHub-W-myrepo` |
| GitHub Team slug | `ag-github-w-<reponame>` | `ag-github-w-myrepo` |
| GitHub Repository | `<reponame>` | `myrepo` |

The script derives the repo name and team slug automatically from the group name.

---

## Prerequisites

### Azure AD
- An Azure AD group following the naming convention `AG-GitHub-W-<reponame>`
- An Azure Service Principal (App Registration) with the following **Application** permissions granted with admin consent:
  - `AppRoleAssignment.ReadWrite.All`
  - `Synchronization.ReadWrite.All`
  - `Group.Read.All`
  - `Directory.ReadWrite.All`

### GitHub
- GitHub organization using **Enterprise Managed Users (EMU)**
- A Classic PAT with `admin:org` and `repo` scopes, **SSO-authorized** for the organization
- The target repository must already exist before running the pipeline

---

## GitHub Actions Secrets

Configure the following secrets in the repository settings under **Settings → Secrets and variables → Actions**:

| Secret | Description |
|--------|-------------|
| `AZURE_CLIENT_ID` | App Registration client ID |
| `AZURE_TENANT_ID` | Azure AD tenant ID |
| `AZURE_CLIENT_SECRET` | App Registration client secret |
| `GH_PAT` | GitHub Classic PAT (`admin:org` + `repo`, SSO-authorized) |

---

## Workflows

### `provision.yml` — Manual trigger
Triggered manually from the GitHub Actions UI. Accepts a group name input and an optional dry-run toggle.

**How to use:**
1. Go to **Actions → GitHub Access Provisioning**
2. Click **Run workflow**
3. Enter the Azure AD group name (e.g. `AG-GitHub-W-myrepo`)
4. Optionally enable **Dry run** to preview without making changes
5. Click **Run workflow**

### `provision-freshdesk.yml` — Freshdesk webhook trigger
Triggered automatically by Freshdesk automation via `repository_dispatch`.

**Freshdesk webhook configuration:**
- **Method:** `POST`
- **URL:** `https://api.github.com/repos/cato-networks-IT/github-access-provisioner/dispatches`
- **Headers:**
  ```
  Accept: application/vnd.github+json
  Content-Type: application/json
  Authorization: token <freshdesk-webhook-pat>
  ```
- **Body:**
  ```json
  {
    "event_type": "provision-github-access",
    "client_payload": {
      "repoName": "{{ticket.ri_214_cf_reponame}}"
    }
  }
  ```

> The Freshdesk webhook PAT only needs `repo` scope — it only triggers the workflow, it does not perform any provisioning itself.

---

## Script Usage

### GitHub Actions (recommended)
The workflow passes the group name directly — no CSV file needed:
```bash
python3 provision-groups-ondemand.py --group "AG-GitHub-W-myrepo" --confirm
```

### Manual / local run
Place a CSV file named `test-ondemand.csv` in the same directory:
```csv
GroupName
AG-GitHub-W-myrepo
```

Then run:
```bash
# Dry run — preview only, no changes
python3 provision-groups-ondemand.py --dry-run

# Real run
python3 provision-groups-ondemand.py --confirm
```

> **Note:** Local runs on macOS behind Cato Networks SASE require the corporate CA certificate. The script handles this automatically when running on macOS.

---

## Pipeline Behaviour

The pipeline is fully **idempotent** — it is safe to run multiple times for the same group:

| Phase | Already done? | Behaviour |
|-------|--------------|-----------|
| Phase 1 — App assignment | Group already assigned | Skipped |
| Phase 1 — SCIM sync | Group already in sync | Treated as success |
| Phase 2 — GitHub Team | Team already exists | Skipped, continues to Phase 3 |
| Phase 3 — Repo permission | Write already assigned | Skipped |

---

## Example Output

```
Full Pipeline: Azure SCIM + GitHub Team + Repo Permission
============================================================
[ Azure ]
  Tenant       : d03fe63f-ee56-4020-a121-dd5b65bc7ea3
[ GitHub ]
  GitHub user  : osher-rachamim_catonian
  GitHub org   : cato-networks (id: 248302678)
  Group        : AG-GitHub-W-myrepo
============================================================

  -- Phase 1: Azure SCIM --
  Object ID          : cd188f85-17c0-4bef-bfae-19071ad6f44b
  App assignment     : Assigned OK
  Waiting 20s for assignment to propagate...
  Triggering on-demand SCIM provisioning...
    ✅ EntryImport: Success
    ✅ EntrySynchronizationScoping: Success
    ✅ EntryExportAdd: Success — Group created in GitHubEnterpriseCloud
  Provisioning       : ✅ Action: Create | Final step: Success

  -- Phase 2: GitHub Team --
  [Team] IDP group found — id: 1464059
  [Team] Created — slug: ag-github-w-myrepo (id: 16728603)
  [Team] IDP group linked ✅

  -- Phase 3: Repo Permission --
  [Repo] Name: myrepo
  [Repo] Exists ✅
  [Repo] Write permission assigned ✅

============================================================
SUMMARY
============================================================
  Phase 1 — SCIM       Assigned (new): 1  |  Provisioned: 1
  Phase 2 — Teams      Created: 1
  Phase 3 — Repo       Assigned: 1
```

---

## Repository Structure

```
github-access-provisioner/
├── provision-groups-ondemand.py       # Main provisioning pipeline script
├── test-ondemand.csv                  # Optional: CSV for manual local testing
├── README.md
└── .github/
    └── workflows/
        ├── provision.yml              # Manual trigger (workflow_dispatch)
        └── provision-freshdesk.yml    # Freshdesk webhook trigger (repository_dispatch)
```
