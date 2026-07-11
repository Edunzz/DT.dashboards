# 📊 Custom Documents Report — Dashboard + Workflow

## What is it?
A dashboard that performs a **census/inventory of the tenant's documents** (dashboards and notebooks): how many exist, when they were created (time series), their distribution by type, and **duplicate detection** by name pattern ("Copy of ...").

The dashboard **does NOT query the API directly**: it reads from a **Grail lookup table** (`/lookups/documents_report`) that is **populated by a workflow** that runs daily. Therefore, the dashboard **only works after** the workflow has been executed at least once.

## Architecture / how it works

```
[Daily Workflow]
   task get_dashboards             -> lists ALL documents via documentsClient.listDocuments (adminAccess)
   task lookups_documents_report   -> converts to JSONL and uploads as lookup table to Grail
                                      (/platform/storage/resource-store/v1/files/tabular/lookup:upload)
        |
        v
[Lookup table /lookups/documents_report]
        |
        v
[Dashboard Custom Documents Report]  -> load "/lookups/documents_report" | ...
```

![Dashboard overview](./img/dashboard-overview.jpeg)

## Files in this folder
- `Custom Documents Report.json` — the dashboard (import in the **Dashboards** app).
- `custom-documents-report.workflow.json` — the workflow (import in the **Workflows** app).
- `img/` — reference screenshots.

## Prerequisites
- Dynatrace SaaS tenant (Grail enabled, **Dashboards** and **Workflows** apps installed).
- Permissions to create/run workflows and to write lookup files in Grail (see Permissions section).

## 🔐 Required permissions for the WORKFLOW

The workflow runs 2 *Run JavaScript* tasks, each requiring different scopes. Configure them in **Account Management** (policies) and also enable them in **Workflows > Settings > Authorization settings** (Primary/Secondary permissions).

**General Workflows / AutomationEngine permissions:**

| Permission (scope) | Purpose |
|---|---|
| `app-engine:apps:run` | List apps and read bundles (base access for Workflows). |
| `app-engine:functions:run` | Use the function-executor (run the Run JavaScript task). |
| `automation:workflows:read` | View workflows. |
| `automation:workflows:write` | Create/edit the workflow and its scheduled trigger. |
| `automation:workflows:run` | Run the workflow manually or on schedule. |

**Task `get_dashboards` — lists documents (uses `documentsClient.listDocuments` with `adminAccess: true`):**

| Permission (scope) | Purpose |
|---|---|
| `document:documents:read` | Read tenant documents. |
| `document:documents:admin` | Required because the code uses `adminAccess: true` to list documents from ALL users. |

**Task `lookups_documents_report` — uploads the lookup table to the Grail Resource Store:**

| Permission (scope) | Purpose |
|---|---|
| `storage:files:write` | Upload/create the lookup table (`lookup:upload` with `overwrite: true`). |
| `storage:files:read` | Read/validate the lookup table in Grail. |
| `storage:files:delete` | (Recommended) enables overwriting an existing table. |

> 💡 The workflow **actor** (user or service user running it) must have ALL these permissions assigned. If any is missing, the task fails with **403 Forbidden**. It is recommended to use a dedicated **service user** with exactly these scopes (principle of least privilege) and restrict `storage:files:*` to the `/lookups/documents_report` prefix when possible.

## ▶️ Installation procedure (step by step)

### Step 1 — Import the dashboard
1. Open the **Dashboards** app in your tenant.
2. Use the **Upload / Import** menu and select `Custom Documents Report.json`.
3. Save. The dashboard will show "no data" until the lookup table exists.

### Step 2 — Import the workflow
1. Open the **Workflows** app.
2. Import `custom-documents-report.workflow.json` (or create a new workflow and paste the 2 *Run JavaScript* tasks).
3. Review the **trigger**: it comes scheduled at `00:00` in the `America/Asuncion` timezone. Adjust it to your preference.

![Workflow with both tasks in Success state](./img/workflow-tasks.jpeg)

### Step 3 — Configure permissions / actor
1. Assign the workflow actor the scopes listed in the **Permissions** section.
2. In **Workflows > Settings > Authorization settings**, enable the listed primary/secondary permissions.

### Step 4 — First run (populate the lookup table)
1. Run the workflow manually (**Run**).
2. Verify that both tasks finish with **OK** status. The `lookups_documents_report` task returns `success: true` and `totalRecords`.
3. Confirm the table was created by running in a Notebook / Investigator:
   ```
   load "/lookups/documents_report" | limit 10
   ```

### Step 5 — View the dashboard
Go back to **Dashboards** and open *Custom Documents Report*. It should now display the count by type, the time series, the donut chart, and the duplicates table.

## ⚙️ Important considerations
- **Dependency order**: the dashboard depends 100% on the lookup table. Without the first workflow execution, all tiles will appear empty.
- **`overwrite: true`**: each run replaces the entire table (lookup tables are fully replaced, no append).
- **Run JavaScript limits**: 120 s timeout, 256 MB RAM, script ≤ ~5 MB. `listDocuments` paginates in chunks of 1000; on tenants with MANY documents, watch the timeout.
- **`adminAccess: true`**: lists documents from all users; that's why it requires `document:documents:admin`. If you only want your own documents, remove it and `document:documents:read` will suffice.
- **Trigger timezone**: defaults to `America/Asuncion` at 00:00; adjust to your region.
- **Lookup path**: `/lookups/documents_report` (follows Grail path rules: starts with `/lookups`, at least two `/`, only alphanumeric, `-`, `_`, `.`, `/`).
- **Dashboard variables**: `$type` (dashboard/notebook), `$timeshift` (1m/1h/1d/7d) and `$documents_copy` (duplicate detection). All loaded from the same lookup.
- **No secrets**: the workflow JSON contains no tokens or sensitive URLs.

## 🖼️ Reference screenshots

> Images in `img/` visually document the dashboard and the workflow.
> Add additional screenshots to that folder and reference them here.

Suggested images:
- `img/dashboard-overview.jpeg` — dashboard overview with real data.
- `img/workflow-tasks.jpeg` — both workflow tasks in Success state.
- `img/permissions.jpeg` — permissions / Authorization settings configuration.
- `img/lookup-verify.jpeg` — lookup table verification in Notebook.

## Credits
Created by **Jose Romero** — jose.romero@dynatrace.com · Dynatrace community contribution.
