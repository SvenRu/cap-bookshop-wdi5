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
3. Fetches an OAuth token from the cTMS UAA (parsed from the `TMS_SERVICE_KEY` secret)
4. Uploads the `.mtar` to cTMS
5. Creates a Transport Request targeting the `GitHubActions-Dev` node with:
   - **namedUser**: the GitHub actor who triggered the workflow
   - **description**: shortened commit SHA + commit message (special characters stripped)

---

## Step 2: Add cTMS Credentials as GitHub Secrets

The workflow needs only **one secret**: the full cTMS service key JSON. Go to **GitHub repo → Settings → Secrets and variables → Actions → New repository secret** and add:

| Secret name | Value |
|---|---|
| `TMS_SERVICE_KEY` | The entire JSON from your cTMS service key |

The workflow extracts `uaa.url`, `uaa.clientid`, `uaa.clientsecret`, and `uri` from the JSON at runtime using `jq`.

The service key JSON looks like this:

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

## Adapting to Your Transport Landscape

The workflow only needs to know the **entry node** — the first node in your cTMS transport route where the artifact is uploaded. cTMS takes care of forwarding it along the configured route from there, regardless of how many nodes your landscape has (DEV → QA, DEV → QA → PRD, or any other shape).

The entry node name is set in one place in `.github/workflows/deploy.yml`:

```bash
\"nodeName\": \"GitHubActions-Dev\"
```

Replace `GitHubActions-Dev` with whatever your entry node is named in **cTMS UI → Landscape Configuration → Transport Nodes**. Node names are case-sensitive and must match exactly.

---

## Customizing the Transport Description

By default the description is auto-generated as `<short-sha> - <commit message>`. There are two ways to override it.

### Option A — One-off: trigger manually with a custom description

Add a `workflow_dispatch` trigger with an optional input to `.github/workflows/deploy.yml`:

```yaml
on:
  push:
    branches:
      - main
  workflow_dispatch:
    inputs:
      description:
        description: 'Transport description'
        required: false
        default: ''
```

Then in the Create Transport Request step, fall back to the auto-generated value when no input is provided:

```bash
COMMIT_MSG=$(git log -1 --format='%s' | tr -cd 'a-zA-Z0-9 \-._~:/?#@!$&()*+,;=%')
DESCRIPTION="${{ github.event.inputs.description }}"
if [ -z "$DESCRIPTION" ]; then
  DESCRIPTION="${GITHUB_SHA::7} - ${COMMIT_MSG}"
fi
```

And use `${DESCRIPTION}` in the `--data` JSON instead of the inline value.

You can then trigger the workflow manually from **GitHub repo → Actions → Build and Transport to cTMS → Run workflow** and type in a custom description.

### Option B — Permanent: edit the workflow file

Simply change the `description` line in `.github/workflows/deploy.yml`:

```bash
\"description\": \"your custom text here\"
```

Any static text or shell variable is valid as long as it only contains characters from the allowed set: Latin letters, numbers, spaces, and `-._~:\/?#[]@!$&()*+,;=%`.

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
POST {uri}/v2/nodes/export
Authorization: Bearer {token}
Content-Type: application/json

{
  "nodeName": "GitHubActions-Dev",
  "contentType": "MTA",
  "storageType": "FILE",
  "entries": [{ "uri": "{fileId}" }],
  "description": "..."
}
```

This creates a Transport Request at the entry node and cTMS forwards it along the configured route. From here, importing and promoting through the landscape is handled in the cTMS UI by the responsible team.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `401 Unauthorized` on token request | Wrong credentials or UAA URL | Check the `TMS_SERVICE_KEY` secret — ensure it is the full service key JSON |
| `404` on `/v2/nodes/export` | Wrong `uri` in service key | Use the `uri` field, not the `uaa.url` |
| `nodeName` not found error | Node name mismatch | Check exact node name spelling in cTMS Landscape Configuration |
| `.mtar` not found | Build output path wrong | Run `mbt build` locally and check where it writes the archive |

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

## Modifying the Transport Creation

The workflow currently uses the **export** endpoint, but depending on your setup you may want the **upload** endpoint instead. Here is the difference:

| | Export (`/v2/nodes/export`) | Upload (`/v2/nodes/upload`) |
|---|---|---|
| **Concept** | Simulate an export from an environment | Directly place a transport in a specific node |
| **`nodeName`** | Source node (e.g. `GitHubActions-Dev`) | Target node (e.g. `GitHubActions-QA`) |
| **Where transport appears** | In the next node in the configured route | In the named node's import queue |
| **When to use** | You build in CI and want cTMS to route from DEV onwards | You want to push directly to a specific node, skipping earlier nodes |

**Use export** when your pipeline represents a DEV build and you want cTMS to own the routing from there.

**Use upload** when you want to inject a transport directly into a specific node regardless of the landscape route — useful for hotfixes or when there is no source node involved.

To switch from export to upload (or vice versa), change a single line in `.github/workflows/deploy.yml`:

```bash
# Export (current default)
--url "${TMS_URI}/v2/nodes/export"

# Upload (direct to a specific node)
--url "${TMS_URI}/v2/nodes/upload"
```

The request body is identical for both endpoints.

---

## Modifying the Transport Description

The description field in the export request accepts Latin letters, numbers, spaces, and `-._~:\/?#[]@!$&()*+,;=%` (max 512 characters). Any other character — such as apostrophes, em dashes, or non-Latin script — will cause cTMS to reject the request. The workflow already strips disallowed characters from the commit message using `tr`.

### Available values from GitHub Actions

You can combine any of the following to build a custom description:

| Value | How to use | Example output |
|---|---|---|
| Short commit SHA | `${GITHUB_SHA::7}` | `a51038f` |
| Full commit SHA | `$GITHUB_SHA` | `a51038f066dd...` |
| Branch name | `$GITHUB_REF_NAME` | `main` |
| Workflow run number | `$GITHUB_RUN_NUMBER` | `42` |
| GitHub actor (triggering user) | `$GITHUB_ACTOR` | `sven-rullmann` |
| Commit subject | `git log -1 --format='%s'` | `fix: correct order logic` |
| Commit author name | `git log -1 --format='%an'` | `Sven Rullmann` |
| Commit author email | `git log -1 --format='%ae'` | `sven@example.com` |
| Commit date | `git log -1 --format='%ai'` | `2026-08-21 10:23:45 +0200` |

### Example descriptions

```bash
# Default: short SHA + commit subject
"${GITHUB_SHA::7} - ${COMMIT_MSG}"

# Include branch and run number
"${GITHUB_REF_NAME} - run ${GITHUB_RUN_NUMBER} - ${COMMIT_MSG}"

# Include actor
"${GITHUB_ACTOR} - ${GITHUB_SHA::7} - ${COMMIT_MSG}"
```

Always pipe the commit message through the sanitizer before using it in the description:

```bash
COMMIT_MSG=$(git log -1 --format='%s' | tr -cd 'a-zA-Z0-9 \-._~:/?#@!$&()*+,;=%')
```

---

## References

- [SAP Cloud Transport Management – API Reference](https://api.sap.com/api/TMS_v2/overview)
- [SAP Help: Using cTMS Programmatically](https://help.sap.com/docs/cloud-transport-management)
- [Project Piper tmsUpload step (archived)](https://github.com/SAP-archive/project-piper-action) — useful for understanding the parameters even though Piper itself is deprecated
- [MTA Build Tool (mbt)](https://sap.github.io/cloud-mta-build-tool/)
