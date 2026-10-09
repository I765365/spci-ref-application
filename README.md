# SPCI Content Operator Skill

A Claude Code skill blueprint for building a **Content Package Import Service** using the SAP SPCI (Service Package Content Import) pattern.

---

## What is Content Provisioning?

Content provisioning in SAP refers to the delivery and installation of application content — configuration data, workflow definitions, rules, or domain records — into a target system. Instead of shipping content hard-coded inside application code, content packages allow content to travel independently as versioned artifacts (ZIP files), which the target service receives, extracts, and imports into its database tables.

The **SPCI pattern** implements this contract:

| Step | What happens |
|---|---|
| **Package** | Content is bundled as a ZIP file containing one JSON file per entity (`Incidents.json`, `Status.json`, …) |
| **Upload** | Client uploads the ZIP via `POST /files` — service stores it on disk and returns a `fileId` |
| **Register** | Client creates a content package via `POST /contentPackages` referencing the `fileId` (LOCAL) or a download URL (REMOTE) |
| **Import** | Service extracts the ZIP asynchronously, maps each JSON file to a DB table, and bulk-inserts the rows |
| **Track** | Client polls `GET /contentPackages/:id/status` until state reaches `SUCCESS` or `ERROR` |

### Key concepts

| Concept | Description |
|---|---|
| **Content Package** | A named, versioned import job. Tracks processing status and links to the source file. |
| **LOCAL mode** | ZIP already uploaded to the service — referenced by `file:<fileId>` |
| **REMOTE mode** | ZIP lives at an HTTPS URL — service downloads it during processing |
| **Tenant isolation** | All data is scoped to a `tenantId` derived from the `sap-sender-tenant-id` request header |
| **Content Resource** | A record created for each entry in the ZIP, cataloguing what was imported |
| **Table map** | A static lookup from JSON filename (without `.json`) to DB table name — must be extended when adding new entity types |

---

## Skill

The `skills/spci-impl/SKILL.md` file is a complete implementation blueprint compatible with the [SAP AI Skills Library](https://skills.cloud.sap). It covers:

- Language-neutral API contract (14 endpoints)
- Database schema for all system and domain tables
- Full handler pseudocode with all edge cases
- Node.js / SAP CAP reference implementation
- Optional two-column data viewer frontend
- Test ZIP generation script
- Full test suite (checklist, negative curls, Jest/supertest)
- Extension guides for new endpoints and entity types

---

## Prerequisites

- [Node.js](https://nodejs.org) ≥ 18
- [@sap/cds-dk](https://www.npmjs.com/package/@sap/cds-dk) installed globally: `npm i -g @sap/cds-dk`
- The generated service project (created by following the skill)

---

## Install

```sh
# 1. Install the skill into Claude Code
npx skills add I765365/spci-content-operator-skills --skill spci-impl

# 2. Ask Claude Code to build the service
# Open Claude Code in your project folder, then type:
#   /spci-impl
```

Claude will scaffold the full service. Once generated, install its dependencies:

```sh
cd <your-project-folder>
npm install
```

---

## Run

```sh
# Start the CAP service (default port 4004)
cds watch

# Or without file-watching:
cds run
```

The service is available at `http://localhost:4004/spci/v1`.

Open the data viewer at `http://localhost:4004/frontend/index.html`.

---

## Quick test

```sh
PORT=4004

# 1. Generate a sample content ZIP
node scripts/create-test-package.mjs

# 2. Upload the ZIP
FILE_ID=$(curl -s -X POST http://localhost:$PORT/spci/v1/files \
  -H "sap-sender-tenant-id: local-tenant" \
  -F "file=@./test-data/sample-package.zip" | jq -r .id)
echo "File ID: $FILE_ID"

# 3. Create a content package (triggers async import)
PKG_ID=$(curl -s -X POST http://localhost:$PORT/spci/v1/contentPackages \
  -H "Content-Type: application/json" \
  -H "sap-sender-tenant-id: local-tenant" \
  -d "{\"name\":\"Test Package\",\"version\":\"1.0.0\",\"type\":\"workflow\",\"location\":{\"type\":\"LOCAL\",\"uri\":\"file:$FILE_ID\"}}" \
  | jq -r .id)
echo "Package ID: $PKG_ID"

# 4. Poll until SUCCESS
curl -s http://localhost:$PORT/spci/v1/contentPackages/$PKG_ID/status \
  -H "sap-sender-tenant-id: local-tenant" | jq .

# 5. Verify imported data
curl -s http://localhost:$PORT/spci/v1/data/Incidents \
  -H "sap-sender-tenant-id: local-tenant" | jq .
```

> **Important:** Use `sap-sender-tenant-id: local-tenant` in all curl commands — this matches the tenant the frontend reads from.

---

## API reference

Base path: `/spci/v1`  
Required header on every request: `sap-sender-tenant-id: <tenantId>` (defaults to `local-tenant`)

| Method | Path | Description |
|---|---|---|
| GET | `/health` | Service health check |
| POST | `/files` | Upload a ZIP file (`multipart/form-data`, field `file`) |
| GET | `/files/:fileId` | Get file metadata |
| DELETE | `/files/:fileId` | Delete file |
| POST | `/contentPackages` | Create package and trigger import |
| GET | `/contentPackages` | List all packages |
| GET | `/contentPackages/:id` | Get one package |
| PUT | `/contentPackages/:id` | Update package and re-trigger import |
| DELETE | `/contentPackages/:id` | Delete package and all its imported data |
| GET | `/contentPackages/:id/status` | Poll processing status |
| GET | `/contentPackages/:id/logs` | Get processing logs |
| GET | `/data/:entity` | Fetch imported rows for a named entity |

---

## Project structure (generated service)

```
<project>/
├── db/
│   └── data-model.cds          # CDS schema — all system + domain entities
├── srv/
│   ├── server.js               # CAP bootstrap + Express wiring
│   └── spci-impl.js            # All 14 route handlers
├── frontend/
│   ├── index.html              # Two-column data viewer
│   ├── api.js                  # Fetch helpers + ENTITIES + COLUMNS config
│   ├── store.js                # In-memory cache
│   └── controller.js           # Polling loop
├── scripts/
│   └── create-test-package.mjs # Generates test-data/sample-package.zip
├── tests/
│   ├── setup.js                # Points CDS at :memory: SQLite
│   ├── health.test.js
│   ├── files.test.js
│   └── packages.test.js
└── uploads/                    # ZIP storage (auto-created, gitignored)
```

---

## Related

- [SAP AI Skills Library](https://skills.cloud.sap)
- [SAP CAP documentation](https://cap.cloud.sap/docs/)
- [SAP BTP Content Provisioning](https://pages.github.tools.sap/atom-cfs/atom-docs/docs/product-concepts/provisioning/about-content-provisioning/)
