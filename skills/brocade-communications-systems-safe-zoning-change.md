---
name: brocade-safe-zoning-change
description: >-
  Make a zoning change on a Brocade Fabric OS switch without causing a storage outage — read the
  current zone database, edit it inside the switch-side transaction, and either activate it or
  abort cleanly.
generated: '2026-09-04'
method: generated
source: >-
  openapi/brocade-communications-systems-zoning-api-openapi.yml,
  openapi/brocade-communications-systems-operations-api-openapi.yml,
  openapi/brocade-communications-systems-session-api-openapi.yml, and
  FOS-92x-REST-API-RM104 (Zoning Operations, Zoning Restrictions, Zoning Transaction Timer
  Restrictions, Zoning Checksums)
api: Brocade Fabric OS REST API
base_url: https://{switch-ip}/rest
operations:
  - login
  - getEffectiveZoning
  - getDefinedZoning
  - patchDefinedZoning
  - zoneOperation
  - logout
---

# Safe zoning change on a Brocade Fabric OS switch

Zoning decides which hosts can see which storage. A wrong change is a storage outage, not a failed
API call. This skill exists to make the change reversible and the failure modes loud.

## Before you start

- You need a Fabric OS account with the `zoneadmin`, `fabricadmin`, `admin` or `securityadmin`
  role. `root` and `maintenance` cannot use the REST API at all.
- Only three REST sessions exist on the switch by default, shared with every other client. Open
  one, do the work, close it.
- Set `Accept: application/yang-data+json` and `Content-Type: application/yang-data+json`, or you
  will get XML — XML is the default.
- If the fabric uses Virtual Fabrics, every call needs the right `vf-id` query parameter. Getting
  this wrong edits the wrong logical switch.

## Steps

### 1. Open a session — `login`

`POST /login` with HTTP Basic credentials. The response `Authorization` header carries the session
token; replay it on every subsequent call. Keep the session for the whole transaction: the zone
transaction is bound to this session ID, and logging out cancels a dangling one.

### 2. Read what is live — `getEffectiveZoning`

`GET /running/brocade-zone/effective-configuration`

This tells you which configuration is currently enforced, and it carries the zone database
checksum and transaction state. Record the checksum. Never skip this step to save a round trip —
the checksum is how the switch stops you overwriting someone else's in-flight change.

### 3. Read what is defined — `getDefinedZoning`

`GET /running/brocade-zone/defined-configuration`

The full zone database: every zone, alias and configuration that exists, enforced or not. Work out
your edit against this, not against the effective configuration.

### 4. Make the edit — `patchDefinedZoning`

`PATCH /running/brocade-zone/defined-configuration`

Send the change with the checksum you read in step 2. A stale checksum fails the operation — that
is correct behaviour, and the right response is to go back to step 2, not to retry blindly.

Constraints that bite here:
- The maximum zone database transaction size is 4 MB. Over that you get `413 Request Entity Too
  Large`; split the change.
- A payload is required when deleting zone members.
- This opens a transaction. Nothing is live yet.

### 5. Decide — `zoneOperation`

`POST /operations/zone`

This is the RPC that activates, disables, saves or **aborts** a zone configuration.

- To commit: activate/enable the configuration.
- To back out: **abort the transaction**. The defined configuration is left as it was.

The abort window is the zone fabric lock timeout — **5 minutes by default**, restarted on each
successive zoning operation in the session. It is configurable, so check the switch rather than
assuming five. After the timeout the transaction can still be recommenced by the original REST
client, but only if no other zone user has started a transaction on the local switch and the
transaction has not been aborted by a zone merge or a remote zone commit.

### 6. Close the session — `logout`

`POST /logout`. This frees the session slot. Note that it also cancels any transaction you left
open, which is a safety net, not a strategy.

## Retry rules

There is no idempotency key on this API. **Do not blind-retry step 4 or step 5.** A retried
`zoneOperation` runs twice.

- If a request does not answer within 30 seconds, Broadcom's own guidance is to retry up to 3
  times — but for a write, re-read the effective configuration first and check whether your change
  already landed.
- On `503 Service Unavailable`, the same retry logic applies.
- In a large fabric, put a one second delay between requests.

## Errors you will actually see

| Status | What it means here |
|---|---|
| 400 | Malformed yang-data body, bad key leaf, or an unencoded reserved character in a query |
| 403 | Your Fabric OS role lacks the Zoning RBAC class |
| 404 | Wrong path, wrong `vf-id`, or a resource deprecated out of this firmware release |
| 413 | Zone database transaction over 4 MB |
| 415 | Accept/Content-Type not set to `application/yang-data+json` |
| 500 | Operation failed; read the application message in the body — it mirrors the CLI error |
| 502 / 503 | Switch busy; back off, retry, do not stack more sessions on it |

There is no `429` and there are no rate-limit headers. Pressure shows up as 502/503 or a failed
login against the session cap.
