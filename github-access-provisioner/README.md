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
  │  POST /repos/cato-networks-IT/github-access-provisioner/dispatches
  │  { "event_type": "provision-github-access", "client_payload": { "repoName": "myrepo" } }
  │
  ▼
GitHub Actions (repository_dispatch)
  │  Generates GitHub App installation token (auto-rotating, no PAT needed)
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

### GitHub App — `cato-github-access-provisioner`
The pipeline uses a **GitHub App** instead of a Personal Access Token for all GitHub operations. This provides:
- Auto-rotating tokens (no manual rotation every 90 days)
- Audit logs show the app name, not a personal account
- Fine-grained permissions scoped only to what is needed
- Higher API rate limits (15,000 req/hour vs 5,000 for PATs)

**The app must be:**
- Created in the `cato-networks` org
- Installed on the `cato-networks` org with **All repositories** access
- Granted the following permissions:
  - Repository → `Administration`: Read & Write
  - Organization → `Members`: Read & Write
  - Repository → `Metadata`: Read (mandatory)

### Freshdesk Webhook PAT
A separate fine-grained PAT is used **only** to trigger the GitHub Actions workflow from Freshdesk. It does not perform any provisioning.
- Scope: `repo` only (to send `repository_dispatch` events)
- Stored as a Freshdesk credential — not in GitHub secrets

---

## GitHub Actions Secrets

Configure the following secrets in **Settings → Secrets and variables → Actions** of this repository:

| Secret | Description | How to obtain |
|--------|-------------|---------------|
| `AZURE_CLIENT_ID` | Azure Service Principal client ID | Azure Portal → App Registrations → your app → Overview |
| `AZURE_TENANT_ID` | Azure AD tenant ID | Azure Portal → Azure Active Directory → Overview |
| `AZURE_CLIENT_SECRET` | Azure Service Principal client secret | Azure Portal → App Registrations → your app → Certificates & secrets |
| `GH_APP_ID` | GitHub App numeric ID | GitHub → org Settings → Developer Settings → GitHub Apps → your app → About → App ID |
| `GH_APP_PRIVATE_KEY` | GitHub App private key (PEM file contents) | GitHub App settings → Private keys → Generate a private key → open the downloaded `.pem` file and paste the full contents including `-----BEGIN RSA PRIVATE KEY-----` and `-----END RSA PRIVATE KEY-----` |

> **Security note:** The `GH_APP_PRIVATE_KEY` secret contains the full PEM file content. GitHub Actions encrypts all secrets at rest. Never commit the `.pem` file to the repository.

---

## Workflows

### `provision-app.yml` — Freshdesk webhook trigger (production)
Triggered automatically by Freshdesk automation via `repository_dispatch`. Uses the GitHub App for authentication.

**How it works:**
1. Freshdesk sends a webhook to GitHub
2. GitHub Actions generates a short-lived installation token from the GitHub App (valid 1 hour, auto-rotated)
3. The token is passed to the Python script as `GH_APP_TOKEN`
4. The full pipeline runs — SCIM, team creation, repo permission

**Freshdesk webhook configuration:**
- **Method:** `POST`
- **URL:** `https://api.github.com/repos/cato-networks-IT/github-access-provisioner/dispatches`
- **Credentials:** Fine-grained PAT stored in Freshdesk credential store (`Authorization: token <pat>`)
- **Headers:**
  ```
  Accept: application/vnd.github+json
  Content-Type: application/json
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
  [Team] Skipping membership removal — running as GitHub App
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
        ├── provision-app.yml          # Freshdesk webhook trigger — GitHub App auth (production)
        ├── provision.yml              # Legacy: manual trigger with PAT
        └── provision-freshdesk.yml    # Legacy: Freshdesk trigger with PAT
```
