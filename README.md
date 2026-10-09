# SPCI Content Operator Skill

A Claude Code skill blueprint for implementing a **ContentRuntime** service using the SAP SAAS Provider Content Interface (SPCI). The generated service acts as the SPCI endpoint that the SAP Content Operator calls to deploy content packages.

---

## What is Content Provisioning?

[SAP Unified Provisioning](https://pages.github.tools.sap/atom-cfs/atom-docs/docs/product-concepts/provisioning/about-content-provisioning/) treats content entities the same way it treats tenants and integrations: as desired-state resources defined in a BTP Blueprint or a `ManagedService`. The **Unified Resource Manager (URM)** manages that desired state and exposes it through a common API.

### The Content Operator

The **Content Operator** provisions content to the content runtime tenant. It watches `ContentPackage` Resource Type Definitions (RTDs) for packages that need to be deployed, and uses the **Service Provider Content Interface (SPCI)** to communicate with the target service.

During reconciliation, the operator compares the desired state of each `ContentPackage` with its current state in the content runtime tenant. If the required content version is not yet present, it triggers deployment via the SPCI endpoint.

```
BTP Blueprint / ManagedService
        │  defines desired state
        ▼
  Unified Resource Manager (URM)
        │  generates ContentPackage resources
        ▼
   Content Operator
        │  calls SPCI API
        ▼
  ContentRuntime  ◄── This is the service this skill builds
           extracts ZIP, imports rows into DB
        

```

### How ContentPackage resources are created

ContentPackage resources can be created in two ways:

1. **`ContentPackageConfig`** — referenced from a `ManagedService`. Useful when one application owns the content definition but it needs to be deployed to a different content runtime tenant (e.g., Central Invoice Management deploying to an Output Management embedded tenant).

2. **BTP Blueprint** — content packages defined in the blueprint are automatically created during provisioning, with metadata supplied by the solution owner.

### Key resource types

| Resource | Description |
|---|---|
| `ContentRuntime` | Describes what your runtime accepts: SPCI endpoint URL, supported content types, content repositories |
| `ContentPackageConfig` | Defines which content packages to deploy and to which target tenant type |
| `ContentTrigger` | Created by `ContentPackageConfig`/Blueprint flows; watched by the Content Leased Operator |
| `ContentPackage` | The reconciled resource that the Content Operator acts on to deploy content |

---

## SPCI API Contract

Your service must implement the following endpoints. The Content Operator calls them in order.

### Storage Management _(skip when `contentAsReference: true`)_

| Method | Path | Description |
|---|---|---|
| `POST /v1/files` | Upload content ZIP | `multipart/form-data`. Returns `201 { "id": "<file-id>" }` |
| `DELETE /v1/files/{file-id}` | Delete content file | Returns `204` |

### Content Package Management

| Method | Path | Description |
|---|---|---|
| `POST /v1/contentPackages` | Import content package | Triggers async or sync import. Returns `202 { "id" }` (async) or `200 { "id" }` (sync) |
| `GET /v1/contentPackages/{id}/status` | Poll import status | Returns `{ "operation", "status", "progress", "lastModified", "details" }` |
| `GET /v1/contentPackages/{id}` | Get package properties | Returns current state of the package |
| `GET /v1/contentPackages/{id}/logs` | Get deployment logs | Returns import log messages |

**Request headers on every call:**

| Header | Value |
|---|---|
| `sap-sender-tenant-id` | Tenant ID of the application that owns the content |
| `sap-receiver-tenant-id` | ID of the ContentRuntime (your service's tenant) |

**POST /contentPackages — request body (LOCAL mode):**

```json
{
  "application": "com.sap.up.contentoperator",
  "originId": "<origin-id>",
  "type": "sap.opm.ads.template",
  "name": "my-application",
  "version": "1.0.0",
  "vendor": "SAP",
  "location": { "type": "LOCAL", "uri": "file:<file-id>" }
}
```

**POST /contentPackages — request body (REMOTE mode):**

```json
{
  "application": "com.sap.up.contentoperator",
  "originId": "<origin-id>",
  "type": "sap.opm.ads.template",
  "name": "my-application",
  "version": "1.0.0",
  "vendor": "SAP",
  "location": { "type": "REMOTE", "uri": "https://example.com/packages/my-app-1.0.0.zip" }
}
```

**GET /contentPackages/:id/status — response shape:**

```json
{
  "operation": "CREATE",
  "lastModified": "<datetime>",
  "status": "RUNNING",
  "progress": "50",
  "details": { "code": "D-001", "message": "Import in progress" }
}
```

Status values: `PENDING` → `RUNNING` → `SUCCESS` | `WARNING` | `ERROR`

---

## Skill

The `skills/spci-impl/SKILL.md` file is a complete implementation blueprint compatible with the [SAP AI Skills Library](https://skills.cloud.sap). It covers:

- Language-neutral API contract (14 endpoints)
- Database schema for all system and domain tables
- Full handler pseudocode with all edge cases
- Node.js / SAP CAP reference implementation
- Optional two-column data viewer frontend
- Test ZIP generation script
- Full test suite (23-item checklist, negative curls, Jest/supertest)
- Extension guides for new endpoints and entity types

---

## Prerequisites

- [Node.js](https://nodejs.org) ≥ 18
- [@sap/cds-dk](https://www.npmjs.com/package/@sap/cds-dk) installed globally

```sh
npm install -g @sap/cds-dk
```

---

## Install the skill

```sh
npx skills add I765365/spci-content-operator-skills --skill spci-impl
```

Then open Claude Code in your project folder and type `/spci-impl` to scaffold the full service.

---

## Install dependencies

```sh
cd <your-project-folder>
npm install
```

---

## Run

```sh
# Start with hot-reload (development)
cds watch

# Start without file-watching (production-like)
cds run
```

Service available at: `http://localhost:4004/spci/v1`  
Data viewer at: `http://localhost:4004/frontend/index.html`

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

# 3. Create a content package — triggers async import
PKG_ID=$(curl -s -X POST http://localhost:$PORT/spci/v1/contentPackages \
  -H "Content-Type: application/json" \
  -H "sap-sender-tenant-id: local-tenant" \
  -d "{\"name\":\"Test Package\",\"version\":\"1.0.0\",\"type\":\"workflow\",\"vendor\":\"SAP\",\"location\":{\"type\":\"LOCAL\",\"uri\":\"file:$FILE_ID\"}}" \
  | jq -r .id)
echo "Package ID: $PKG_ID"

# 4. Poll until SUCCESS
curl -s http://localhost:$PORT/spci/v1/contentPackages/$PKG_ID/status \
  -H "sap-sender-tenant-id: local-tenant" | jq .

# 5. Verify imported data
curl -s http://localhost:$PORT/spci/v1/data/Incidents \
  -H "sap-sender-tenant-id: local-tenant" | jq .
```

> **Note:** Use `sap-sender-tenant-id: local-tenant` in all curl commands — this matches the tenant the frontend reads from.

---

## Project structure (generated)

```
<project>/
├── db/
│   └── data-model.cds          # CDS schema — system + domain entities
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

## Related links

- [About Content Provisioning — atom-docs](https://pages.github.tools.sap/atom-cfs/atom-docs/docs/product-concepts/provisioning/about-content-provisioning/)
- [SPCI API spec](https://pages.github.tools.sap/content-propagation-service/cps-api/api/spci.html)
- [SAP AI Skills Library](https://skills.cloud.sap)
