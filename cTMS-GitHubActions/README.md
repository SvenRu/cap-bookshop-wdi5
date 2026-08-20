# CI/CD to Production: Automating SAP Cloud Transport Management with GitHub Actions

> **Audience:** BTP extension developers who want to replace manual transport steps with a fully automated pipeline.
> **What you'll build:** A GitHub Actions workflow that builds your MTA, uploads it to cTMS, and triggers the transport to QA and PRD — no Piper required.

---

## Background

SAP Cloud Transport Management (cTMS) is the gatekeeper between your development artifact and your production tenant. Traditionally, teams leaned on [Project Piper](https://www.project-piper.io/) to bridge their CI/CD pipeline with cTMS. Piper is now in **deprecation mode** (archived July 2026), so new projects should call the cTMS REST API directly.

The good news: the API is straightforward, and GitHub Actions makes it easy to script the whole flow with plain `curl` calls and a handful of secrets.

---

## What We're Building

```
Developer git push
        │
        ▼
  GitHub Actions
  ┌─────────────────────────────────────────┐
  │ 1. Build MTA archive (.mtar)            │
  │ 2. Fetch OAuth token from cTMS UAA      │
  │ 3. Upload .mtar → cTMS (file upload)    │
  │ 4. Create Transport Request in cTMS     │
  └─────────────────────────────────────────┘
        │
        ▼ (cTMS takes over)
   QA Tenant  →  PRD Tenant
```

GitHub Actions owns steps 1–4. From there, cTMS manages the transport route you've configured: it deploys to QA, and once that passes, promotes to PRD.

---

## Prerequisites

Before you start, you need the following in place:

1. **A BTP subaccount** with the SAP Cloud Transport Management service subscribed and a service instance created.
   - BTP Cockpit → Service Marketplace → Cloud Transport Management → Create instance
2. **A service key** for that cTMS instance.
   - BTP Cockpit → your cTMS instance → Service Keys → Create
   - Keep the downloaded JSON — you'll need four values from it in Step 2.
3. **A transport landscape configured in cTMS** with nodes and routes (DEV → QA → PRD).
   - See Step 3 below for the exact setup.
4. **An MTA project** with a valid `mta.yaml` (this sample app already has one).

---

## Step 1: Enable GitHub Actions in Your Repository

Create the workflow file `.github/workflows/deploy.yml` in your repository. The file is already provided in this project at that path. It does the following on every push to `main`:

1. Checks out the code and installs dependencies
2. Builds the MTA archive with `mbt`
3. Fetches an OAuth token from the cTMS UAA
4. Uploads the `.mtar` to cTMS
5. Creates a Transport Request targeting the `DEV` node

---

## Step 2: Add cTMS Credentials as GitHub Secrets

The workflow reads cTMS connection details from GitHub repository secrets. Go to **GitHub repo → Settings → Secrets and variables → Actions → New repository secret** and add:

| Secret name | Value from service key |
|---|---|
| `CTMS_UAA_URL` | `uaa.url` |
| `CTMS_CLIENT_ID` | `uaa.clientid` |
| `CTMS_CLIENT_SECRET` | `uaa.clientsecret` |
| `CTMS_API_URL` | `uri` |

You find these values in the cTMS service key JSON from BTP:

```json
{
  "uaa": {
    "clientid": "sb-...",
    "clientsecret": "...",
    "url": "https://<subaccount>.authentication.<region>.hana.ondemand.com"
  },
  "uri": "https://transport-service.cfapps.<region>.hana.ondemand.com"
}
```

> **Tip:** Never commit service key JSON to your repo. If you do so by accident, rotate the secret immediately in BTP.

---

## Step 3: Configure Your Transport Landscape in cTMS

Before the pipeline can push anything, cTMS needs to know the shape of your landscape.

1. Open the **cTMS UI** (the URL from your service key's `uri` field).
2. Go to **Landscape Configuration → Transport Nodes** and create:
   - `GitHubActions-Dev` — your development tenant (source, no import queue needed if you deploy directly here from CI)
   - `GitHubActions-QA` — your QA tenant (with import queue enabled)
   - `GitHubActions-Prd` — your production tenant (with import queue enabled)
3. Go to **Transport Routes** and create:
   - `GitHubActions-Dev → GitHubActions-QA`
   - `GitHubActions-QA → GitHubActions-Prd`

When your GitHub Actions workflow uploads an artifact and targets the `GitHubActions-Dev` node, cTMS will forward it along this route automatically.

---

## Step 4: Trigger the First Workflow Run

To verify everything is wired up correctly, make a minimal change and push to `main`. A good candidate is adding a CI/CD note to the root `README.md` — something that is genuinely useful and serves as the trigger:

```bash
git add README.md
git commit -m "chore: add CI/CD transport management section"
git push origin main
```

Then go to **GitHub repo → Actions tab** and watch the `Build and Transport to cTMS` workflow run. On success you will see:

- A green checkmark on the Actions run
- A job summary showing the **File ID** and **Transport Request ID** from cTMS
- A new transport request visible in the cTMS UI under the `GitHubActions-Dev` import queue

---

## What Each Step Does

### Step 3 — OAuth Token

cTMS uses **OAuth 2.0 Client Credentials** flow. You POST to the UAA token endpoint with your `clientid` and `clientsecret`. The returned `access_token` is a short-lived JWT (typically 12 hours) that authorizes all subsequent API calls.

```
POST {uaa.url}/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id={clientid}
&client_secret={clientsecret}
```

### Step 4 — File Upload

```
POST {uri}/v2/files/upload
Authorization: Bearer {token}
Content-Type: multipart/form-data

file: <binary .mtar>
```

Returns a `fileId` — a reference to the binary stored in cTMS. The file is not yet associated with any transport route.

### Step 5 — Create Transport Request

```
POST {uri}/v2/nodes/upload
Authorization: Bearer {token}
Content-Type: application/json

{
  "nodeName": "DEV",        ← must match a node name in your landscape
  "contentType": "MTA",
  "storageType": "FILE",
  "entries": [{ "uri": "{fileId}" }],
  "description": "..."
}
```

This creates a Transport Request and queues the uploaded file for import into the `DEV` node's import queue. cTMS then forwards it along your configured route to QA and PRD.

---

## Step 4: Handling the QA → PRD Promotion

By default, cTMS requires a **manual import trigger** between nodes. This is intentional — it's your approval gate before production.

You have two options:

**Option A — Manual (recommended for most teams)**
A release manager logs into the cTMS UI and clicks **Import** on the transport request in the QA node. After validation, they repeat for PRD.

**Option B — Automated via API (for mature pipelines)**
Add another GitHub Actions job (or a separate `workflow_dispatch` workflow) that calls:

```bash
# Trigger import into a specific node
POST {uri}/v2/nodes/{nodeId}/importRequests
Authorization: Bearer {token}
Content-Type: application/json

{
  "transportRequests": ["{transportRequestId}"]
}
```

This is useful for QA if you have automated tests gating the PRD promotion, but **use with caution for PRD** — automated deploys to production require strong confidence in your test coverage.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `401 Unauthorized` on file upload | Expired or missing token | Check UAA URL and credentials; ensure `grant_type=client_credentials` |
| `404` on `/v2/nodes/upload` | Wrong `CTMS_API_URL` | Use the `uri` field from the service key, not the UAA URL |
| `nodeName` not found error | Node name mismatch | Check exact node name spelling in cTMS Landscape Configuration |
| `.mtar` not found | Build output path wrong | Run `mbt build` locally and check where it writes the archive |
| Transport stuck in import queue | No auto-import configured | Trigger manually in cTMS UI or use the importRequests API |

---

## Security Considerations

- **Rotate service keys** periodically in BTP and update GitHub secrets accordingly.
- **Restrict branch triggers** — only run this workflow on `main` (or your release branch), not on every feature branch.
- **Use GitHub Environments** with required reviewers for the PRD import job if you go fully automated.
- **Do not log the token** — the workflow above uses `$GITHUB_OUTPUT` which is masked in logs. Avoid `echo`-ing the token directly.

---

## Summary

| Step | Responsibility |
|---|---|
| Build `.mtar` | GitHub Actions (mbt) |
| Fetch OAuth token | GitHub Actions → cTMS UAA |
| Upload archive | GitHub Actions → cTMS API `/v2/files/upload` |
| Create Transport Request | GitHub Actions → cTMS API `/v2/nodes/upload` |
| Import to QA | cTMS (manual or API trigger) |
| Promote to PRD | cTMS (manual or API trigger) |

With this setup, every push to `main` results in a ready-to-import transport request in cTMS — your infrastructure team keeps control of when it lands in QA and PRD, while developers never touch the transport process manually.

---

## References

- [SAP Cloud Transport Management – API Reference](https://api.sap.com/api/TMS_v2/overview)
- [SAP Help: Using cTMS Programmatically](https://help.sap.com/docs/cloud-transport-management)
- [Project Piper tmsUpload step (archived)](https://github.com/SAP-archive/project-piper-action) — useful for understanding the parameters even though Piper itself is deprecated
- [MTA Build Tool (mbt)](https://sap.github.io/cloud-mta-build-tool/)
