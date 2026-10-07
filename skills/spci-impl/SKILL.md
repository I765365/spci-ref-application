---
name: spci-impl
description: Complete blueprint for building a content-package import service (SPCI pattern) in any language. Covers the language-neutral API contract, database schema, library substitution guide, a Node.js/SAP CAP reference implementation, an optional two-column data-viewer frontend, test ZIP generation, and extension guides.
category: Development
tags: [CAP, Node.js, REST, SAP BTP, ZIP import, content package, SPCI]
trigger: spci, content package import, ZIP import service, SPCI pattern
---

You are helping build a **Content Package Import Service** (the SPCI pattern) in any language.

The service lets external systems push ZIP-based content packages via REST. The backend extracts the ZIP and imports each JSON file inside it as rows into a matching database table. The frontend is a **read-only data viewer** — it polls the backend and renders the imported rows as live tables. All writes come from curl or API clients.

Read this skill top-to-bottom before writing any code. Sections 1–5 are language-agnostic. Section 6 is the Node.js/SAP CAP reference implementation — use it only when building in Node.js, otherwise use sections 1–5 as the complete spec.

---

## 1. Core concepts

| Concept | Description |
|---|---|
| **File upload** | Client uploads a ZIP via `multipart/form-data`. Backend stores it on disk, records metadata in DB, returns a `fileId`. |
| **Content package** | A named, versioned import job. Points to a ZIP either by `fileId` (LOCAL) or by a download URL (REMOTE). |
| **Async processing** | After the HTTP response is sent, the backend extracts the ZIP and imports each `<EntityName>.json` file into the matching DB table. Status moves `INITIAL → PROCESSING → SUCCESS/ERROR`. |
| **Table map** | A lookup from JSON filename (no extension) to DB table name. Must be updated whenever a new importable entity is added. |
| **Tenant isolation** | Every DB read/write is scoped to a `tenantId` derived from the `sap-sender-tenant-id` request header. |
| **Data viewer endpoint** | `GET /data/:entity` returns all rows for a named entity so the frontend can render them. |

---

## 2. Language-neutral API contract

All endpoints live under the base path `/spci/v1`. The tenant header `sap-sender-tenant-id` must be present on every request (fall back to `'local-tenant'` if missing).

### Health

| | |
|---|---|
| `GET /health` | Returns `{ "status": "OK", "timestamp": "<iso>" }` with HTTP 200. |

### Files

| Method | Path | Purpose |
|---|---|---|
| `POST /files` | Upload a ZIP | `multipart/form-data`, field name `file`. Returns `201 { "id": "<fileId>" }`. |
| `GET /files/:fileId` | Get file metadata | Returns `200 { "id", "uri", "checksumAlgorithm", "checksumValue", "processingStatus" }` or `404`. |
| `DELETE /files/:fileId` | Delete file | Removes disk file + DB record. Returns `204`. |

### Content packages

| Method | Path | Purpose |
|---|---|---|
| `POST /contentPackages` | Create / upsert package | Body: `{ name, version, type, location: { type: "LOCAL"\|"REMOTE", uri } }`. Returns `201 { "id" }`. Triggers async processing. |
| `GET /contentPackages` | List all packages | Returns `200 [{ id, name, version, status, ... }]`. |
| `GET /contentPackages/:id` | Get one package | Returns `200 { id, name, version, status, ... }` or `404`. |
| `PUT /contentPackages/:id` | Update package | Same body shape as POST. Returns `200 { "id" }`. Re-triggers processing. |
| `DELETE /contentPackages/:id` | Delete package | Removes package, its logs, its content resources, and all imported domain rows. Returns `204`. |
| `GET /contentPackages/:id/status` | Poll processing status | Returns `200 { operation, state, progress, lastModified, details }`. |
| `GET /contentPackages/:id/logs` | Get processing logs | Returns `200 { id, messages: [{ id, createdAt, message, severity }] }`. |

### Data viewer

| Method | Path | Purpose |
|---|---|---|
| `GET /data/:entity` | Fetch imported rows | Returns `200 { entity, count, rows: [{...}] }` or `404` for unknown entity. |

### Error response shape (all errors)

```json
{ "error": { "code": "<string>", "message": "<string>", "details": [] } }
```

Error code ranges: `1xxx` file errors · `2xxx` package errors · `3xxx` processing errors · `9xxx` general errors.

### Location URI rules

| `location.type` | `contentAsReference` header | Expected `location.uri` format |
|---|---|---|
| `LOCAL` | false / absent | `file:<uuid>` (the fileId from a prior upload) |
| `LOCAL` | true | A valid `https://` URL |
| `REMOTE` | any | A valid `https://` URL |

---

## 3. Database schema (language-neutral)

Implement these five tables in whichever DB/ORM your language uses. Column names are shown in camelCase; adjust to your convention.

### `file_metadata`
| Column | Type | Notes |
|---|---|---|
| `fileId` | string PK | UUID |
| `fileName` | string | Original filename |
| `filePath` | string | Absolute path on disk |
| `fileSize` | integer | Bytes |
| `mimeType` | string | |
| `checksum` | string | SHA-256 or UUID placeholder |
| `uploadedAt` | datetime | |
| `tenantId` | string | Always filter by this |

### `content_package`
| Column | Type | Notes |
|---|---|---|
| `contentPackageId` | string PK | UUID |
| `name` | string | |
| `description` | string | |
| `version` | string | |
| `fileId` | string | FK → file_metadata (nullable for REMOTE) |
| `metadata` | string | JSON-encoded request body extras |
| `status` | string | `INITIAL` \| `PROCESSING` \| `SUCCESS` \| `WARNING` \| `ERROR` |
| `createdAt` | datetime | |
| `updatedAt` | datetime | |
| `tenantId` | string | Always filter by this |

### `content_package_log`
| Column | Type | Notes |
|---|---|---|
| `logId` | string PK | UUID |
| `contentPackageId` | string | FK → content_package |
| `timestamp` | datetime | |
| `message` | string | |
| `severity` | string | `INFO` \| `WARNING` \| `ERROR` \| `FATAL` |

### `content_resource`
| Column | Type | Notes |
|---|---|---|
| `contentID` | string PK | UUID |
| `type` | string | From package `type` field |
| `version` | string | From package `version` field |
| `packageId` | string | FK → content_package |
| `vendor` | string | From package metadata `vendor` field |
| `status` | string | Always `"Active"` |
| `uri` | string | `/content/<packageId>/<entryName>` |
| `storageType` | string | From metadata `application` field, default `"SPCI"` |

### Domain tables (one per importable entity)
Add one table per JSON file type you want to import. The table name must match the value in your **table map**. Columns come directly from the JSON file's field names — inspect the test ZIP to define them.

---

## 4. Complete handler pseudocode

Implement every handler below exactly as specified. Edge cases are called out explicitly — do not skip them.

---

### POST /files — uploadFile

```
function uploadFile(request):
  file = request.multipart_file("file")

  // 1. Missing file
  if file is null:
    return 400 { error: { code: "1001", message: "file is required",
                           details: [{ code: "1006", message: "file is required" }] } }

  // 2. Must be a ZIP — check extension AND mime type
  extension = lowercase(file.original_filename.extension)   // e.g. ".zip"
  mime      = file.mime_type
  isZip     = extension == ".zip"
           OR mime == "application/zip"
           OR mime == "application/x-zip-compressed"
  if NOT isZip:
    delete_temp_file(file.temp_path)
    return 400 { error: { code: "1002", message: "Only ZIP files are allowed",
                           details: [{ code: "1007", message: "File must be a ZIP archive. Received: <ext or mime>" }] } }

  // 3. Size limit: 50 MB
  MAX = 50 * 1024 * 1024
  if file.size > MAX:
    delete_temp_file(file.temp_path)
    return 400 { error: { code: "1003", message: "File size exceeds 50MB",
                           details: [{ code: "1008", message: "File size: <X>MB, Maximum: 50MB" }] } }

  // 4. Persist to disk with a UUID filename
  fileId      = new_uuid()
  newFileName = fileId + extension                    // e.g. "<uuid>.zip"
  newPath     = join("uploads", newFileName)
  move(file.temp_path, newPath)

  // 5. Write metadata to DB
  db.insert("file_metadata", {
    fileId:     fileId,
    fileName:   file.original_filename,
    filePath:   newPath,
    fileSize:   file.size,
    mimeType:   mime,
    checksum:   new_uuid(),                           // placeholder; replace with SHA-256 if needed
    uploadedAt: now_iso(),
    tenantId:   request.tenantId
  })

  return 201 { "id": fileId }
```

---

### GET /files/:fileId — getFile

```
function getFile(request):
  fileId = request.path_param("fileId")
  row    = db.select_one("file_metadata", where: { fileId, tenantId: request.tenantId })

  if row is null:
    return 404 { error: { code: "1004", message: "The specified resource was not found" } }

  return 200 {
    id:                 row.fileId,
    uri:                "file:" + row.fileId,
    checksumAlgorithm:  "SHA-256",
    checksumValue:      row.checksum,
    processingStatus: {
      state:        "OK",
      lastModified: row.uploadedAt,
      progress:     "100%",
      details:      { code: "MSG001", message: "Successfully finished" }
    }
  }
```

---

### DELETE /files/:fileId — deleteFile

```
function deleteFile(request):
  fileId = request.path_param("fileId")
  row    = db.select_one("file_metadata", where: { fileId, tenantId: request.tenantId })

  if row is null:
    return 404 { error: { code: "1004", message: "The specified resource was not found" } }

  delete_file_from_disk(row.filePath)   // ignore error if file already gone
  db.delete("file_metadata", where: { fileId, tenantId: request.tenantId })
  return 204
```

---

### POST /contentPackages — createContentPackage

```
function createContentPackage(request):
  body    = parse_json(request.body)
  name    = body.name
  version = body.version
  type    = body.type
  location = body.location
  contentAsReference = request.header("contentAsReference") == "true"

  // 1. Required fields
  if NOT name OR NOT version OR NOT type:
    return 400 { error: { code: "1001", message: "name, version, and type are required", details: [...] } }

  if NOT location:
    return 400 { error: { code: "2004", message: "location is required", details: [...] } }

  // 2. location.type must be LOCAL or REMOTE (case-insensitive)
  locationType = uppercase(location.type)
  if locationType NOT IN ["LOCAL", "REMOTE"]:
    return 400 { error: { code: "2003", message: "location.type must be LOCAL or REMOTE", details: [...] } }

  // 3. location.uri is required
  if NOT location.uri:
    return 400 { error: { code: "2005", message: "location.uri is required", details: [...] } }

  // 4. Validate URI format
  expectUrl  = locationType == "REMOTE" OR contentAsReference == true
  expectUuid = locationType == "LOCAL"  AND contentAsReference == false

  if expectUrl AND location.uri does NOT match /^https?:\/\/.+/:
    return 400 { error: { code: "2010", message: "location.uri must be a valid URL", details: [...] } }

  if expectUuid AND location.uri does NOT match UUID or "file:<uuid>":
    return 400 { error: { code: "2010", message: "location.uri must be a UUID or file:<uuid>", details: [...] } }

  // 5. Upsert — if (name, version, tenantId) already exists, update and re-process
  existing = db.select_one("content_package",
               where: { name, version, tenantId: request.tenantId })

  if existing:
    packageId = existing.contentPackageId
    updateData = { status: "INITIAL", updatedAt: now_iso() }
    if locationType == "REMOTE":  updateData.metadata = serialize(body)
    else:                         updateData.fileId = strip_prefix(location.uri, "file:")
    db.update("content_package", updateData, where: { contentPackageId: packageId })
    send 200 { "id": packageId }          // respond BEFORE processing starts
    fire_and_forget process(packageId, locationType, location.uri, request)
    return

  // 6. New package
  packageId = new_uuid()
  fileId    = null
  contentUri = null
  if locationType == "REMOTE":
    contentUri = location.uri
  else:
    fileId = strip_prefix(location.uri, "file:")   // "file:abc-123" → "abc-123"

  db.insert("content_package", {
    contentPackageId: packageId,
    name, version, type,
    description:  body.attributes?.description ?? "",
    fileId,
    metadata:     serialize(body),                  // store full request body as JSON string
    status:       "INITIAL",
    createdAt:    now_iso(),
    updatedAt:    now_iso(),
    tenantId:     request.tenantId
  })

  send 201 { "id": packageId }            // respond BEFORE processing starts
  fire_and_forget process(packageId, locationType, location.uri, request)
```

---

### processContentPackage (LOCAL mode — async, no HTTP response)

```
function processContentPackage(packageId, tenantId, db):
  logs = []

  try:
    db.update("content_package", { status: "PROCESSING", updatedAt: now_iso() },
               where: { contentPackageId: packageId })

    pkg  = db.select_one("content_package", where: { contentPackageId: packageId })
    file = db.select_one("file_metadata",   where: { fileId: pkg.fileId })

    if file is null:
      logs.append("ERROR", "File metadata not found")
      db.update("content_package", { status: "ERROR", updatedAt: now_iso() }, ...)
      save_logs(packageId, logs, db)
      return

    logs.append("INFO", "Processing file: " + file.fileName)
    zip      = open_zip(file.filePath)
    metadata = parse_json(pkg.metadata)
    type     = metadata.type ?? "unknown"
    vendor   = metadata.vendor ?? "Unknown"
    processed = 0
    imported  = 0

    for entry in zip.entries():
      if entry.is_directory:
        continue
      if entry.name.starts_with("_"):         // skip manifest/meta files
        continue

      // Create a ContentResource record for every non-directory entry
      resourceId = new_uuid()
      db.insert("content_resource", {
        contentID:   resourceId,
        type,
        version:     pkg.version,
        packageId,
        vendor,
        status:      "Active",
        uri:         "/content/" + packageId + "/" + entry.name,
        storageType: metadata.application ?? "SPCI"
      })
      processed++
      logs.append("INFO", "Created ContentResource for " + entry.name)

      // Import JSON data into domain table
      if entry.name.ends_with(".json"):
        entityName = entry.name.remove_suffix(".json")   // "Incidents.json" → "Incidents"
        tableName  = tableMap[entityName]                 // lookup

        if tableName is null:
          logs.append("WARNING", "No table mapping for " + entityName + ", skipping")
          continue

        try:
          rows = parse_json(entry.read_text())
          if NOT is_array(rows) OR rows is empty:
            continue
          logs.append("INFO", "Importing " + len(rows) + " records into " + entityName)
          for row in rows:
            try:
              db.insert(tableName, row)
              imported++
            catch UNIQUE_CONSTRAINT_ERROR:
              logs.append("INFO", "Record already exists in " + entityName + ", skipping")
            catch other_error:
              logs.append("WARNING", "Failed to insert record in " + entityName + ": " + error.message)
          logs.append("INFO", "Finished importing " + entityName)
        catch parse_error:
          logs.append("WARNING", "Failed to parse JSON from " + entry.name + ": " + parse_error.message)

    logs.append("INFO", "Completed. Created " + processed + " resources, imported " + imported + " records.")
    db.update("content_package", { status: "SUCCESS", updatedAt: now_iso() }, ...)

  catch error:
    logs.append("ERROR", "Processing error: " + error.message)
    db.update("content_package", { status: "ERROR", updatedAt: now_iso() }, ...)

  finally:
    save_logs(packageId, logs, db)
```

---

### processContentPackageReference (REMOTE mode — async, no HTTP response)

```
function processContentPackageReference(packageId, contentUri, repoAccessToken, tenantId, db):
  logs = []

  try:
    db.update("content_package", { status: "PROCESSING", updatedAt: now_iso() }, ...)
    logs.append("INFO", "Starting REMOTE processing. URI: " + contentUri)

    // 1. Validate URI
    if NOT contentUri.starts_with("http://") AND NOT contentUri.starts_with("https://"):
      logs.append("ERROR", "Invalid URI: must start with http:// or https://")
      db.update("content_package", { status: "ERROR", ... }, ...)
      save_logs(packageId, logs, db)
      return

    // 2. Download
    headers = { "Accept": "application/vnd.github.v3.raw", "User-Agent": "SPCI-Service" }
    if repoAccessToken:
      headers["Authorization"] = "token " + repoAccessToken
      logs.append("INFO", "Using provided repoAccessToken")
    else if env["GITHUB_TOKEN"]:
      headers["Authorization"] = "token " + env["GITHUB_TOKEN"]
      logs.append("INFO", "Using GITHUB_TOKEN from environment")
    else:
      logs.append("WARNING", "No auth token — request may fail for private repos")

    response = http_get(contentUri, headers, follow_redirects: true)
    if response.status != 200:
      logs.append("ERROR", "Download failed: HTTP " + response.status)
      db.update("content_package", { status: "ERROR", ... }, ...)
      save_logs(packageId, logs, db)
      return

    // 3. Save downloaded bytes as a ZIP file
    fileId   = new_uuid()
    filePath = join("uploads", fileId + ".zip")
    write_bytes(filePath, response.body_bytes)
    logs.append("INFO", "Downloaded " + len(response.body_bytes) + " bytes → " + filePath)

    // 4. Process exactly like LOCAL mode from here
    // (same ZIP loop, same tableMap, same ContentResource creation, same import logic)
    pkg      = db.select_one("content_package", where: { contentPackageId: packageId })
    metadata = parse_json(pkg.metadata)
    type     = metadata.type ?? "unknown"
    vendor   = metadata.vendor ?? "Unknown"
    zip      = open_zip(filePath)
    processed = 0
    imported  = 0

    for entry in zip.entries():
      if entry.is_directory OR entry.name.starts_with("_"):
        continue
      resourceId = new_uuid()
      db.insert("content_resource", { contentID: resourceId, type, version: pkg.version,
                  packageId, vendor, status: "Active",
                  uri: "/content/" + packageId + "/" + entry.name,
                  storageType: metadata.application ?? "SPCI" })
      processed++
      if entry.name.ends_with(".json"):
        entityName = entry.name.remove_suffix(".json")
        tableName  = tableMap[entityName]
        if tableName:
          rows = parse_json(entry.read_text())
          for row in rows:
            try:   db.insert(tableName, row); imported++
            catch UNIQUE_CONSTRAINT_ERROR: logs.append("INFO", "Duplicate skipped in " + entityName)
            catch err: logs.append("WARNING", "Insert failed in " + entityName + ": " + err.message)

    // 5. Store fileId reference on the package so it can be traced back
    db.update("content_package", {
      status: "SUCCESS", fileId: fileId, updatedAt: now_iso()
    }, where: { contentPackageId: packageId })
    logs.append("INFO", "Completed. Resources: " + processed + ", records: " + imported)

  catch error:
    logs.append("ERROR", "Processing error: " + error.message)
    db.update("content_package", { status: "ERROR", updatedAt: now_iso() }, ...)

  finally:
    save_logs(packageId, logs, db)
```

---

### saveLogs

```
function save_logs(packageId, log_entries, db):
  // log_entries = list of (severity, message) tuples collected during processing
  records = []
  for (severity, message) in log_entries:
    records.append({
      logId:            new_uuid(),
      contentPackageId: packageId,
      timestamp:        now_iso(),
      message:          message,
      severity:         severity    // "INFO" | "WARNING" | "ERROR" | "FATAL"
    })
  if records is not empty:
    db.bulk_insert("content_package_log", records)
```

---

### GET /contentPackages/:id/status — getContentPackageStatus

```
STATUS_MAP   = { INITIAL: "PENDING", PROCESSING: "RUNNING",
                 SUCCESS: "SUCCESS", WARNING: "WARNING", ERROR: "ERROR" }
PROGRESS_MAP = { INITIAL: "0%", PROCESSING: "50%",
                 SUCCESS: "100%", WARNING: "100%", ERROR: "100%" }
MESSAGE_MAP  = { INITIAL: "Initialized", PROCESSING: "In progress",
                 SUCCESS: "Successfully finished",
                 WARNING: "Completed with warnings", ERROR: "Processing failed" }

function getContentPackageStatus(request):
  id  = request.path_param("id")
  pkg = db.select_one("content_package",
          columns: ["status", "updatedAt"],
          where:   { contentPackageId: id, tenantId: request.tenantId })

  if pkg is null:
    return 404 { error: { code: "2002", message: "The specified resource was not found" } }

  status = pkg.status
  return 200 {
    operation:    "CREATE",
    state:        STATUS_MAP[status]   ?? "PENDING",
    progress:     PROGRESS_MAP[status] ?? "0%",
    lastModified: pkg.updatedAt,
    details: {
      code:    "CEC-002",
      message: MESSAGE_MAP[status] ?? "Processing"
    }
  }
```

---

### DELETE /contentPackages/:id — deleteContentPackage

```
function deleteContentPackage(request):
  id       = request.path_param("id")
  existing = db.select_one("content_package",
               where: { contentPackageId: id, tenantId: request.tenantId })

  if existing is null:
    return 404 { error: { code: "2002", message: "The specified resource was not found" } }

  // Cannot delete while processing
  if existing.status == "PROCESSING":
    return 422 { error: { code: "2007", message: "Cannot delete package while it is being processed" } }

  // Delete all imported domain rows (add one line per importable entity)
  db.delete("domain_table_1")     // e.g. Incidents
  db.delete("domain_table_2")     // e.g. Customers
  // ...

  db.delete("content_resource",     where: { packageId: id })
  db.delete("content_package_log",  where: { contentPackageId: id })
  db.delete("content_package",      where: { contentPackageId: id, tenantId: request.tenantId })
  return 204
```

---

### GET /data/:entity — getEntityData

```
function getEntityData(request):
  entity    = request.path_param("entity")
  tableName = VIEWABLE_ENTITIES[entity]   // same map as tableMap

  if tableName is null:
    return 404 { error: { code: "2009",
      message: "Unknown entity: " + entity + ". Available: " + join(VIEWABLE_ENTITIES.keys, ", ") } }

  rows = db.select_all(tableName)
  return 200 { entity, count: len(rows), rows }
```

---

## 5. Library substitution guide

Choose one library per role from the table below. The pattern (upload → store → extract → import → respond) is identical regardless of language.

| Role | Node.js (reference) | Python | Java | Go |
|---|---|---|---|---|
| HTTP framework | `express` | `FastAPI` / `Flask` | `Spring Boot` | `net/http` / `gin` |
| File upload / multipart | `multer` | `python-multipart` | `MultipartFile` (Spring) | `r.FormFile` (stdlib) |
| ZIP extraction | `adm-zip` | `zipfile` (stdlib) | `java.util.zip` | `archive/zip` (stdlib) |
| Database ORM / driver | `@sap/cds` + SQLite/HANA | `SQLAlchemy` / `sqlite3` | `Spring Data JPA` / `JDBC` | `database/sql` + `modernc.org/sqlite` |
| UUID generation | `uuid` | `uuid` (stdlib) | `java.util.UUID` | `google/uuid` |
| HTTP client (REMOTE mode) | `node-fetch` | `httpx` / `requests` | `RestTemplate` / `WebClient` | `net/http` (stdlib) |
| Auth / JWT | `@sap/xssec` + `passport` | `python-jose` / `PyJWT` | `spring-security-oauth2` | `golang-jwt/jwt` |

**Processing pattern (pseudocode — implement in your language):**

```
function processPackage(packageId, tenantId):
  update status → PROCESSING
  file = fetchFileMetadata(packageId, tenantId)
  zip  = openZip(file.path)                    // or download from URI for REMOTE

  for entry in zip.entries():
    if entry.isDirectory or entry.name.startsWith("_"):
      continue
    createContentResource(entry.name, packageId)
    if entry.name.endsWith(".json"):
      rows = parseJsonArray(entry.data)
      table = tableMap[entry.nameWithoutExtension]
      if table:
        bulkInsert(table, rows)                // skip duplicates gracefully

  update status → SUCCESS
  saveLogs(packageId, logMessages)
```

**Tenant isolation pattern:**

```
// Every request handler — read tenant from header, scope all DB calls to it
tenantId = request.header("sap-sender-tenant-id") ?? "local-tenant"
db.query("SELECT * FROM content_package WHERE tenantId = ?", tenantId)
```

---

## 6. Node.js / SAP CAP reference implementation

> Use this section only when building in Node.js. For other languages, sections 1–5 are the complete spec.

### npm dependencies

```json
"dependencies": {
  "@sap/cds": "^7",
  "adm-zip": "^0.5",
  "express": "^4",
  "multer": "^1.4.5-lts.1",
  "node-fetch": "^2",
  "passport": "^0.7",
  "@sap/xssec": "^3",
  "@sap/xsenv": "^5",
  "uuid": "^11"
}
```

### CDS schema (db/data-model.cds)

```cds
namespace consarea.db;

entity FileMetadata {
  key fileId     : String;
      fileName   : String;
      filePath   : String;
      fileSize   : Integer;
      mimeType   : String;
      checksum   : String;
      uploadedAt : DateTime;
      tenantId   : String;
}

entity ContentPackage {
  key contentPackageId : String;
      name             : String;
      description      : String;
      version          : String;
      fileId           : String;
      metadata         : String;
      status           : String;
      createdAt        : DateTime;
      updatedAt        : DateTime;
      tenantId         : String;
}

entity ContentPackageLog {
  key logId            : String;
      contentPackageId : String;
      timestamp        : DateTime;
      message          : String;
      severity         : String;
}

entity ContentResource {
  key contentID   : String;
      type        : String;
      version     : String;
      packageId   : String;
      vendor      : String;
      status      : String;
      uri         : String;
      storageType : String;
}

// Domain entities — columns match test ZIP field names; adjust for your use case
entity Incidents {
  key ID           : String;
      title        : String;
      urgency_code : String;
      status_code  : String;
      customer_ID  : String;
}

entity Customers {
  key ID        : String;
      firstName : String;
      lastName  : String;
      email     : String;
      phone     : String;
}

entity Incidents_conversation {
  key ID        : String;
      timestamp : DateTime;
      author    : String;
      message   : String;
}

entity Status {
  key code        : String;
      description : String;
}

entity Urgency {
  key code        : String;
      description : String;
}
```

CDS table name = replace `.` with `_`: `consarea.db.ContentPackage` → `consarea_db_ContentPackage`.

### Backend skeleton (srv/spci-impl.js)

```js
import Router from 'express';
import multer from 'multer';
import fs from 'fs';
import cds from '@sap/cds';
import AdmZip from 'adm-zip';
import { v4 as uuidv4 } from 'uuid';
import path from 'path';

const Logger = cds.log('spci-service');
const { SELECT, INSERT, UPDATE, DELETE } = cds.ql;

const MTLS_ENABLED    = false;
const SPCI_AUTH_ENABLED = false;

// Keep in sync with tableMap in both processing methods
const VIEWABLE_ENTITIES = {
  Incidents:               'consarea_db_Incidents',
  Customers:               'consarea_db_Customers',
  IncidentsConverstations: 'consarea_db_Incidents_conversation',
  Status:                  'consarea_db_Status',
  Urgency:                 'consarea_db_Urgency',
};

class SPCIImpl {
  constructor() {
    this._router = Router();
    this._router.use(this.setTenantId.bind(this));
    this.fileUploader = multer({ dest: 'uploads' }).single('file');

    this._router.get('/health', (req, res) =>
      res.status(200).json({ status: 'OK', timestamp: new Date().toISOString() })
    );
    this._router.get('/data/:entity', this.getEntityData.bind(this));

    this._router.post('/files', this.checkFileField.bind(this), this.uploadFile.bind(this));
    this._router.get('/files/:fileId', this.getFile.bind(this));
    this._router.delete('/files/:fileId', this.deleteFile.bind(this));

    this._router.post('/contentPackages', this.createContentPackage.bind(this));
    this._router.get('/contentPackages', this.getAllContentPackages.bind(this));
    this._router.get('/contentPackages/:id', this.getContentPackage.bind(this));
    this._router.put('/contentPackages/:id', this.updateContentPackage.bind(this));
    this._router.delete('/contentPackages/:id', this.deleteContentPackage.bind(this));
    this._router.get('/contentPackages/:id/status', this.getContentPackageStatus.bind(this));
    this._router.get('/contentPackages/:id/logs', this.getContentPackageLogs.bind(this));
  }

  getRouter() { return this._router; }
}
export { SPCIImpl };
```

### Tenant middleware

```js
setTenantId(req, res, next) {
  req.senderTenantId = req.headers['sap-sender-tenant-id']
    || req.authInfo?.getSubaccountId()
    || 'local-tenant';
  req.receiverTenantId = req.headers['sap-receiver-tenant-id'];
  req.tenantId         = req.receiverTenantId || req.senderTenantId;
  req.contentAsReference = req.get('contentAsReference') === 'true';
  req.repoAccessToken    = req.get('repoAccessToken');
  next();
}
```

### Handler pattern

```js
async myHandler(req, res) {
  const id = req.params.id;
  try {
    cds.context = { tenant: req.tenantId };
    const db  = await cds.connect.to('db');
    const row = await db.run(
      SELECT.one.from('consarea_db_ContentPackage')
        .where({ contentPackageId: id, tenantId: req.tenantId })
    );
    // HANA returns UPPERCASE field names; SQLite returns camelCase — handle both
    const name = row.name || row.NAME;
    res.status(200).json({ id, name });
  } catch (err) {
    Logger.error(`Error in myHandler for ${id}:`, err);
    res.status(500).json({ error: { code: '9001', message: err.message, details: [] } });
  }
}
```

### packageView helper

`GET /contentPackages` and `GET /contentPackages/:id` must return `metadata`, `type`, and `fileId` so the frontend can display the original API parameters. Implement a view helper that includes them:

```js
function packageView(r) {
  const raw = r.METADATA || r.metadata || '{}';
  let parsed = {};
  try { parsed = JSON.parse(raw); } catch (_) {}
  return {
    id:          r.CONTENTPACKAGEID || r.contentPackageId,
    name:        r.NAME        || r.name,
    version:     r.VERSION     || r.version,
    description: r.DESCRIPTION || r.description,
    type:        r.TYPE        || r.type        || parsed.type,
    fileId:      r.FILEID      || r.fileId,
    status:      r.STATUS      || r.status,
    createdAt:   r.CREATEDAT   || r.createdAt,
    updatedAt:   r.UPDATEDAT   || r.updatedAt,
    metadata:    raw,           // raw JSON string of original request body
  };
}
```

> HANA returns UPPERCASE column names; SQLite returns camelCase — the `r.FIELD || r.field` pattern handles both.

### Data viewer endpoint

```js
async getEntityData(req, res) {
  const { entity } = req.params;
  const tableName  = VIEWABLE_ENTITIES[entity];
  if (!tableName) {
    return res.status(404).json({
      error: { code: '2009', message: `Unknown entity: ${entity}. Available: ${Object.keys(VIEWABLE_ENTITIES).join(', ')}` }
    });
  }
  try {
    cds.context = { tenant: req.tenantId };
    const db   = await cds.connect.to('db');
    const rows = await db.run(SELECT.from(tableName));
    res.status(200).json({ entity, count: rows.length, rows });
  } catch (err) {
    Logger.error(`Error fetching ${entity}:`, err);
    res.status(500).json({ error: { code: '9001', message: err.message } });
  }
}
```

### Processing modes (fire-and-forget)

```js
// After sending the HTTP response — never await these
this.processContentPackage(packageId, tenantId, db);                        // LOCAL
this.processContentPackageReference(packageId, uri, token, tenantId, db);   // REMOTE
```

Both set status → `PROCESSING` immediately, then `SUCCESS`/`ERROR`, calling `saveLogs()` throughout.

### Table map (keep identical in BOTH processing methods)

```js
const tableMap = {
  'Incidents':               'consarea_db_Incidents',
  'Customers':               'consarea_db_Customers',
  'IncidentsConverstations': 'consarea_db_Incidents_conversation',
  'Status':                  'consarea_db_Status',
  'Urgency':                 'consarea_db_Urgency',
};
```

ZIP rules: files starting with `_` are skipped; non-JSON create a `ContentResource` only; JSON must be a top-level array `[{...}]`.

### Wiring into server.js

```js
import { SPCIImpl } from './spci-impl.js';
const spciService = new SPCIImpl();

cds.on('bootstrap', async (app) => {
  app.use(express.json());
  app.use(express.urlencoded({ extended: true }));
  app.use('/spci/v1', spciService.getRouter());
});
```

---

## 7. Creating test content ZIP files

Generate a ZIP that matches your domain entities before running any curls.

**scripts/create-test-package.mjs** (Node.js — adapt for your language using stdlib ZIP tools)

```js
import AdmZip from 'adm-zip';
import fs from 'fs';

const zip = new AdmZip();

zip.addFile('Incidents.json', Buffer.from(JSON.stringify([
  { ID: 'INC-001', title: 'Login failure',  urgency_code: 'H', status_code: 'N', customer_ID: 'CUST-001' },
  { ID: 'INC-002', title: 'Slow dashboard', urgency_code: 'M', status_code: 'A', customer_ID: 'CUST-002' },
  { ID: 'INC-003', title: 'Missing export', urgency_code: 'L', status_code: 'R', customer_ID: 'CUST-001' },
])));

zip.addFile('Customers.json', Buffer.from(JSON.stringify([
  { ID: 'CUST-001', firstName: 'Alice', lastName: 'Smith', email: 'alice@example.com', phone: '+1-555-0101' },
  { ID: 'CUST-002', firstName: 'Bob',   lastName: 'Jones', email: 'bob@example.com',   phone: '+1-555-0102' },
])));

zip.addFile('Status.json', Buffer.from(JSON.stringify([
  { code: 'N', description: 'New' },
  { code: 'A', description: 'In Process' },
  { code: 'R', description: 'Resolved' },
  { code: 'C', description: 'Closed' },
])));

zip.addFile('Urgency.json', Buffer.from(JSON.stringify([
  { code: 'H', description: 'High' },
  { code: 'M', description: 'Medium' },
  { code: 'L', description: 'Low' },
])));

// Prefix _ = skipped by importer, safe for metadata
zip.addFile('_manifest.json', Buffer.from(JSON.stringify({
  name: 'Test Package', version: '1.0.0', type: 'workflow', vendor: 'ACME',
})));

fs.mkdirSync('test-data', { recursive: true });
zip.writeZip('test-data/sample-package.zip');
console.log('Created test-data/sample-package.zip');
```

For Python: use `zipfile.ZipFile` + `json.dumps`. For Java: use `java.util.zip.ZipOutputStream`. For Go: use `archive/zip`.

---

## 8. Frontend — data viewer (language-agnostic)

> **Before building this section, ask the user:** "Do you want a read-only data viewer frontend? (yes / no)"
> Skip this entire section if they say no — the service works fully without it via curl.

The frontend is vanilla HTML/JS. It requires two backend endpoints:
- `GET /spci/v1/contentPackages` — shows the API parameters the package was created with
- `GET /spci/v1/data/:entity` — shows the imported rows

The UI is a **two-column layout**:
- **Left panel** — "API Parameters" table: one row per field of the content package (name, version, type, location type, URI, file ID, status, timestamps). Populated from the `metadata` field returned by `GET /contentPackages`.
- **Right panel** — a single data table for the active entity. Tabs appear only when `ENTITIES` has more than one entry.

**Column control:** `COLUMNS` in `api.js` is an optional per-entity whitelist. Omit an entity to show all its columns; add it with a list of keys to show only those.

### Store (frontend/store.js)

```js
const cache    = new Map();  // entity → { columns, rows }
let   packages = [];         // content package list

export const Store = {
  set:         (entity, data) => cache.set(entity, data),
  get:         (entity)       => cache.get(entity) || { columns: [], rows: [] },
  entities:    ()             => [...cache.keys()],
  setPackages: (list)         => { packages = list; },
  getPackages: ()             => packages,
};
```

### API client (frontend/api.js)

```js
const BASE    = '/spci/v1';
const HEADERS = { 'sap-sender-tenant-id': 'local-tenant' };

// Add entity names here — must match VIEWABLE_ENTITIES keys in the backend
export const ENTITIES = ['Incidents'];

// Optional column whitelist per entity — omit an entity to show all its columns
export const COLUMNS = {
  Incidents: ['ID', 'title'],
};

export async function fetchEntity(entity) {
  const r = await fetch(`${BASE}/data/${entity}`, { headers: HEADERS });
  if (!r.ok) return { entity, count: 0, rows: [] };
  return r.json();
}

export async function fetchPackages() {
  const r = await fetch(`${BASE}/contentPackages`, { headers: HEADERS });
  if (!r.ok) return [];
  return r.json();
}
```

### Controller (frontend/controller.js)

```js
import { Store }                        from './store.js';
import { ENTITIES, fetchEntity, fetchPackages } from './api.js';

export async function refresh() {
  const [pkgs] = await Promise.all([
    fetchPackages(),
    ...ENTITIES.map(entity =>
      fetchEntity(entity).then(({ rows }) =>
        Store.set(entity, { columns: rows.length ? Object.keys(rows[0]) : [], rows })
      )
    ),
  ]);
  Store.setPackages(pkgs);
}

export function startPolling(onRefresh, intervalMs = 3000) {
  refresh().then(onRefresh);
  return setInterval(() => refresh().then(onRefresh), intervalMs);
}
```

### Page (frontend/index.html)

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>SPCI Data Viewer</title>
  <style>
    *, *::before, *::after { box-sizing: border-box; }
    body   { font-family: sans-serif; margin: 2rem; background: #f4f6f8; color: #333; }
    h1     { margin-bottom: .25rem; }
    #status { font-size: .8rem; color: #888; margin: .25rem 0 1.25rem; }

    /* Two-column layout */
    .layout { display: grid; grid-template-columns: 300px 1fr; gap: 1.5rem; align-items: start; }

    /* Left — API params panel */
    .pkg-panel { background: #fff; border-radius: 6px; box-shadow: 0 1px 3px rgba(0,0,0,.1); padding: 1rem 1.25rem; }
    .pkg-panel h2 { margin: 0 0 .75rem; font-size: 1rem; color: #0070f3;
                    border-bottom: 2px solid #0070f3; padding-bottom: .3rem; }
    .pkg-name  { font-weight: 600; font-size: .95rem; margin-bottom: .6rem; }
    .param-table { width: 100%; border-collapse: collapse; font-size: .82rem; }
    .param-key   { color: #888; white-space: nowrap; padding: .25rem .5rem .25rem 0;
                   vertical-align: top; width: 45%; }
    .param-val   { word-break: break-all; padding: .25rem 0; vertical-align: top; }
    .pkg-divider { border: none; border-top: 1px solid #eee; margin: .75rem 0; }
    .badge {
      display: inline-block; padding: .15rem .55rem; border-radius: 3px;
      font-size: .75rem; font-weight: 600; text-transform: uppercase;
    }
    .badge-SUCCESS    { background: #e6f4ea; color: #1e7e34; }
    .badge-PROCESSING { background: #fff3cd; color: #856404; }
    .badge-ERROR      { background: #fde8e8; color: #c62828; }
    .badge-INITIAL    { background: #e8eaf6; color: #3949ab; }

    /* Right — tabs + single table */
    .tabs { display: flex; gap: .25rem; }
    .tab  {
      padding: .45rem 1.1rem; cursor: pointer; border: 1px solid #d0d7de;
      border-bottom: none; border-radius: 4px 4px 0 0; background: #edf0f3;
      font-size: .85rem; color: #555; user-select: none;
    }
    .tab.active { background: #fff; color: #0070f3; font-weight: 600; }
    table  { width: 100%; border-collapse: collapse; background: #fff;
             box-shadow: 0 1px 3px rgba(0,0,0,.1); }
    th     { background: #0070f3; color: #fff; padding: .55rem 1rem;
             text-align: left; font-size: .85rem; white-space: nowrap; }
    td     { padding: .45rem 1rem; border-bottom: 1px solid #eee; font-size: .85rem; }
    tr:hover td { background: #f0f7ff; }
    .empty { color: #999; font-style: italic; padding: .75rem 1rem; }
  </style>
</head>
<body>
  <h1>SPCI Data Viewer</h1>
  <div id="status">Connecting…</div>

  <div class="layout">
    <div id="pkg-panel" class="pkg-panel"><p style="color:#999;font-size:.85rem">Loading…</p></div>
    <div>
      <div class="tabs" id="tabs"></div>
      <div id="table-wrap"></div>
    </div>
  </div>

  <script type="module">
    import { Store }             from './store.js';
    import { ENTITIES, COLUMNS } from './api.js';
    import { startPolling }      from './controller.js';

    let activeEntity = ENTITIES[0];

    function visibleColumns(entity, allColumns) {
      return COLUMNS[entity] ? COLUMNS[entity].filter(c => allColumns.includes(c)) : allColumns;
    }

    function statusBadge(s) {
      return `<span class="badge badge-${s ?? 'INITIAL'}">${s ?? 'INITIAL'}</span>`;
    }

    function renderPanel() {
      const pkgs  = Store.getPackages();
      const panel = document.getElementById('pkg-panel');
      if (!pkgs.length) {
        panel.innerHTML = '<h2>API Parameters</h2><p style="color:#999;font-size:.85rem">No packages yet.</p>';
        return;
      }
      panel.innerHTML = '<h2>API Parameters</h2>' + pkgs.map((p, i) => {
        const meta = (() => { try { return JSON.parse(p.metadata ?? '{}'); } catch { return {}; } })();
        const loc  = meta.location ?? {};
        const rows = [
          ['Status',    statusBadge(p.status)],
          ['Name',      p.name    ?? '—'],
          ['Version',   p.version ?? '—'],
          ['Type',      p.type    ?? meta.type ?? '—'],
          ['Loc. Type', loc.type  ?? '—'],
          ['URI',       loc.uri   ?? '—'],
          ['File ID',   p.fileId  ?? '—'],
          ['Created',   p.createdAt ? new Date(p.createdAt).toLocaleString() : '—'],
          ['Updated',   p.updatedAt ? new Date(p.updatedAt).toLocaleString() : '—'],
        ];
        return `
          ${i > 0 ? '<hr class="pkg-divider">' : ''}
          <div class="pkg-name">${p.name ?? '—'} <span style="font-weight:400;color:#888">v${p.version ?? '—'}</span></div>
          <table class="param-table">
            <tbody>${rows.map(([k, v]) =>
              `<tr><td class="param-key">${k}</td><td class="param-val">${v}</td></tr>`
            ).join('')}</tbody>
          </table>`;
      }).join('');
    }

    function renderTabs() {
      if (ENTITIES.length < 2) { document.getElementById('tabs').innerHTML = ''; return; }
      document.getElementById('tabs').innerHTML = ENTITIES.map(e =>
        `<div class="tab${e === activeEntity ? ' active' : ''}" data-entity="${e}">
           ${e} (${Store.get(e).rows.length})
         </div>`
      ).join('');
      document.querySelectorAll('.tab').forEach(t =>
        t.addEventListener('click', () => { activeEntity = t.dataset.entity; renderAll(); })
      );
    }

    function renderTable() {
      const { columns, rows } = Store.get(activeEntity);
      const cols = visibleColumns(activeEntity, columns);
      if (!rows.length) {
        document.getElementById('table-wrap').innerHTML =
          `<table><tr><td class="empty">No data yet — import a content package to populate ${activeEntity}.</td></tr></table>`;
        return;
      }
      document.getElementById('table-wrap').innerHTML = `
        <table>
          <thead><tr>${cols.map(c => `<th>${c}</th>`).join('')}</tr></thead>
          <tbody>${rows.map(row =>
            `<tr>${cols.map(c => `<td>${row[c] ?? ''}</td>`).join('')}</tr>`
          ).join('')}</tbody>
        </table>`;
    }

    function renderAll() {
      renderPanel();
      renderTabs();
      renderTable();
      document.getElementById('status').textContent =
        `Last refreshed: ${new Date().toLocaleTimeString()} — auto-refreshes every 3 s`;
    }

    startPolling(renderAll);
  </script>
</body>
</html>
```

**How it works:**
- Left panel reads `metadata` (the raw JSON request body stored on the package) to show exactly what was sent in the API call — type, location type, URI, file ID, status badge, timestamps.
- Right panel shows a single table; tabs appear automatically when `ENTITIES` has more than one entry.
- `COLUMNS` in `api.js` is a per-entity column whitelist — omit an entity to show all columns, add it to restrict to specific fields.
- All fetches use `sap-sender-tenant-id: local-tenant` — ensure your curl commands use the same header so the frontend and data are in the same tenant.

---

## 9. Testing the full flow

> **Important:** Use `sap-sender-tenant-id: local-tenant` in all curl commands — this matches the tenant the frontend uses, so imported data appears in the viewer immediately.

```sh
# 1. Generate test ZIP (Node.js shown — adapt for your language)
node scripts/create-test-package.mjs

# 2. Upload ZIP
curl -X POST http://localhost:<port>/spci/v1/files \
  -H "sap-sender-tenant-id: local-tenant" \
  -F "file=@./test-data/sample-package.zip"
# → { "id": "<fileId>" }

# 3. Create package (triggers async import)
curl -X POST http://localhost:<port>/spci/v1/contentPackages \
  -H "Content-Type: application/json" \
  -H "sap-sender-tenant-id: local-tenant" \
  -d '{"name":"Test Package","version":"1.0.0","type":"workflow","location":{"type":"LOCAL","uri":"file:<fileId>"}}'
# → { "id": "<pkgId>" }

# 4. Poll until SUCCESS
curl http://localhost:<port>/spci/v1/contentPackages/<pkgId>/status \
  -H "sap-sender-tenant-id: local-tenant"
# → { "state": "SUCCESS", "progress": "100%", ... }

# 5. Confirm data was imported
curl http://localhost:<port>/spci/v1/data/Incidents \
  -H "sap-sender-tenant-id: local-tenant"
# → { "entity": "Incidents", "count": 3, "rows": [...] }

# 6. Open the viewer (if frontend was built)
# http://localhost:<port>/frontend/index.html
# Left panel shows API parameters; right panel shows the Incidents table
```

REMOTE mode:
```sh
curl -X POST http://localhost:<port>/spci/v1/contentPackages \
  -H "Content-Type: application/json" \
  -H "sap-sender-tenant-id: local-tenant" \
  -H "repoAccessToken: <token>" \
  -d '{"name":"Remote","version":"1.0.0","type":"workflow","location":{"type":"REMOTE","uri":"https://.../.zip"}}'
```

---

## 10. Adding a new endpoint

1. Register the route (before `getRouter()` / equivalent in your framework).
2. Implement the handler — always scope DB queries to `tenantId`, always return `{ error: { code, message, details } }` on failure.
3. File-upload routes need the multipart middleware applied before the handler.

## 11. Adding a new importable data type

1. Add the DB table / entity to your schema.
2. Add `'<FileName>': '<tableName>'` to **both** the table map (in both processing methods) and to the viewable-entities lookup.
3. Add a matching `zip.addFile('<FileName>.json', ...)` entry in the test ZIP script.
4. Add `'<FileName>'` to `ENTITIES` in `frontend/api.js` — the table appears automatically.
5. If rows should be removed on package delete, add a `DELETE FROM <tableName>` call in the delete handler.
