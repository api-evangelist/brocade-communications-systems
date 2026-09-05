---
name: brocade-fabric-inventory
description: >-
  Take a read-only inventory and health snapshot of a Brocade Fabric OS switch — chassis, switch
  attributes, Fibre Channel ports and per-port error statistics — without changing anything.
generated: '2026-09-04'
method: generated
source: >-
  openapi/brocade-communications-systems-chassis-api-openapi.yml,
  openapi/brocade-communications-systems-switch-api-openapi.yml,
  openapi/brocade-communications-systems-interfaces-api-openapi.yml,
  openapi/brocade-communications-systems-session-api-openapi.yml, and FOS-92x-REST-API-RM104
api: Brocade Fabric OS REST API
base_url: https://{switch-ip}/rest
operations:
  - login
  - getChassis
  - getSwitch
  - getFcInterfaces
  - getFcInterfaceStatistics
  - logout
---

# Read-only fabric inventory from a Brocade switch

Every operation in this skill is a GET. Nothing here changes switch state, which makes it the safe
place to start when you are learning a fabric.

## Two ways to run it

**Session-less** (Fabric OS 9.1.0 and later, GET only) is the better fit for a one-shot inventory:
one request performs login, the GET and logout. Use HTTP Basic — plain text (`curl -u`) or
`Authorization: Basic <base64>`. Session-less requests do not consume a session slot, but at least
one slot must be free for them to run.

**Session-based** is better when you are pulling many resources: one `login`, many GETs, one
`logout`, so you pay the authentication cost once.

## Steps

1. **`login`** — `POST /login`, HTTP Basic. Take the token from the `Authorization` response
   header. Skip this entirely if you are going session-less.
2. **`getChassis`** — `GET /running/brocade-chassis/chassis`. The enclosure: chassis WWN, name,
   serial, part number, licence identifier. One per managed device.
3. **`getSwitch`** — `GET /running/brocade-fibrechannel-switch/fibrechannel-switch`. Switch WWN,
   domain ID, fabric name, enabled state, principal-switch role. With Virtual Fabrics there is one
   of these per logical switch — iterate `vf-id`.
4. **`getFcInterfaces`** — `GET /running/brocade-interface/fibrechannel`. Every Fibre Channel port,
   keyed by `slot/port` (e.g. `4/5`): speed, operational status, port type, attached device WWN.
5. **`getFcInterfaceStatistics`** — `GET /running/brocade-interface/fibrechannel-statistics`.
   Per-port counters and error statistics. Join to step 4 on the `name` key.
6. **`logout`** — `POST /logout` if you opened a session.

## Reading the result

- The join key across all of it is the **WWN** (`20:04:00:11:0d:75:00:09`) and the `slot/port`
  name. If you put a WWN in a query parameter, percent-encode the colons — `$ & + , / : ; = ? @`
  are reserved under RFC 3986 and the API rejects them raw.
- Statistics are cumulative counters. A single sample tells you almost nothing; two samples with a
  known interval tell you a rate. Space them out — the pacing rules below still apply to reads.
- Confirm what you are talking to: the resource API version comes back as a Content-Type
  parameter (`;version=2.0.0`), and per-module versions come from `GET /rest/brocade-module-version`.

## Pacing

This is a management plane on a switch that is also moving storage traffic.

- Three FOS REST API client instances per fabric is Broadcom's recommendation, and SANnav and
  Network Advisor are already using two of the fabric's budget between them.
- Expect a response in under 30 seconds. If you do not get one, retry up to 3 times.
- Add a one second delay between requests in a large fabric. Reads are safe to retry — there is no
  idempotency mechanism on this API, but a GET has nothing to duplicate.
- No pagination exists. A container URI returns the whole list.

## Scope note

The six modules in this profile are 6 of 41. A fuller inventory would also read the name server,
FDMI, MAPS dashboards, licensing, firmware and FRU modules. They exist in the API; they are just
not captured here.
