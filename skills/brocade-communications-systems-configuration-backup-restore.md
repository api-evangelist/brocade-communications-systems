---
name: brocade-configuration-backup-restore
description: >-
  Capture a Brocade Fabric OS switch configuration before making changes, poll the asynchronous
  operation to completion, and restore it if a change goes wrong.
generated: '2026-09-04'
method: generated
source: >-
  openapi/brocade-communications-systems-operations-api-openapi.yml,
  openapi/brocade-communications-systems-session-api-openapi.yml, and FOS-92x-REST-API-RM104
  (Configuration Upload and Download, brocade-operation-configupload, brocade-operation-configdownload)
api: Brocade Fabric OS REST API
base_url: https://{switch-ip}/rest
operations:
  - login
  - configUpload
  - showStatus
  - configDownload
  - logout
---

# Back up and restore a Brocade Fabric OS switch configuration

This API has no undo. What it has is a save-and-restore pair, and the discipline of using it
*before* you change anything is the difference between a bad afternoon and a bad quarter.

## Steps

### 1. `login`

`POST /login` with HTTP Basic credentials; carry the `Authorization` token forward. You need an
`admin`-class Fabric OS role — `mgmtapp --show rbacinfo` on the switch shows the RBAC class each
URL demands.

### 2. Take the backup — `configUpload`

`POST /operations/configupload`

Uploads the running configuration from the switch to a remote host you specify in the request
body. This is the "before" state. Do this every time, before any `patchSwitch`, `patchChassis`,
`patchDefinedZoning` or firmware operation.

The RPC is asynchronous: the response carries a **`message-id`**.

### 3. Poll to completion — `showStatus`

`POST /operations/show-status/message-id/<message-id>`

Poll until the operation reports completion. Do not assume the 200 on step 2 means the file
landed — it means the switch accepted the request.

Pacing: expect under 30 seconds for an individual request, retry up to 3 times if nothing comes
back, and leave a second between polls in a large fabric.

### 4. Verify before you trust it

Confirm the file exists on the remote host and is the size you expect. A backup you have not
verified is not a backup.

### 5. Make your change

Whatever it was — zoning, switch attributes, chassis configuration. If it was zoning, use the
`brocade-safe-zoning-change` skill instead: zoning has a real transaction and a real abort, and
you should use those rather than falling back to a configuration restore.

### 6. Restore if needed — `configDownload`

`POST /operations/configdownload`

Applies a saved configuration to the switch. Also asynchronous — poll `showStatus` the same way.

**No restore window is published.** Broadcom documents the mechanism, not a deadline, so treat the
backup as valid until the fabric has drifted from it, and take a fresh one before every change
rather than relying on yesterday's.

### 7. `logout`

`POST /logout` to release the session slot.

## What you cannot take back

- `brocade-operation-factory-reset` — irreversible over the API. Recovery means a previously
  uploaded configuration and a maintenance window.
- `brocade-operation-firmwaredownload` — no REST-level rollback is published. It does support
  downloading to a secondary partition without rebooting or activating, which is the closest thing
  to a rehearsal this API offers. Use it.
- `brocade-operation-reboot` — there is no idempotency key on this API. A retried reboot reboots
  twice.

## The rule that matters

There is no `Idempotency-Key` header, no deduplication window, and no replay protection anywhere
on this surface. The retry guidance Broadcom publishes — three attempts on a 30-second timeout or
a 503 — is written for reads. For a mutating operation, re-check state before you retry. For
zoning specifically, use the transaction; it is the only place on this API where "undo" is a real
operation.
