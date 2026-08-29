---
name: logicmonitor-triage-alerts
description: >-
  Pull the live LogicMonitor alert queue, narrow it to what matters, acknowledge with a note, and record
  what you did as an Ops Note.
api: LogicMonitor REST API v3
base_url: https://{account}.logicmonitor.com/santaba/rest
spec: openapi/logicmonitor-rest-api-v3-swagger.json
generated: '2026-08-29'
method: generated
source: Grounded in operationIds and x-minimum-permissions read from the published v3 Swagger.
operations:
  - getAlertList
  - getAlertById
  - getAlertListByDeviceId
  - ackAlertById
  - addAlertNoteById
  - addOpsNote
permissions:
  - 'Resources: View'
  - 'Resources: Acknowledge'
  - 'Settings -> Ops Notes: Manage'
---

# Triage the LogicMonitor alert queue

## 1. Pull the queue

`getAlertList` — `GET /alert/alerts?filter=severity>:3,cleared:false&size=200&fields=id,severity,monitorObjectName,dataPointName,startEpoch,acked,sdted`

Severity is numeric: **2 = warning, 3 = error, 4 = critical**. Use `fields` to keep the payload small —
the Alert schema is wide and you rarely need all of it.

This endpoint has its own rate limit of **400/min**, below the 500/min GET default. Paginate with
`size` and `offset`; there is no cursor and no next-link, so you increment `offset` yourself and stop when
`items` comes back shorter than `size`.

## 2. Drop the noise before you read it

- `sdted: true` means the alert is inside a scheduled downtime — someone already declared this expected.
- `acked: true` means it is already owned.
- To scope to one resource instead: `getAlertListByDeviceId` — `GET /device/devices/{id}/alerts`, which
  has a **higher** limit of 600/min.

## 3. Read one alert in full

`getAlertById` — `GET /alert/alerts/{id}`

`monitorObjectName` and `instanceName` tell you what broke, `dataPointName` and `alertValue` tell you the
measurement that tripped, `rule` and `chain` tell you where the notification went.

## 4. Acknowledge with a reason

`ackAlertById` — `POST /alert/alerts/{id}/ack` with an `ackComment`.

**This is one-way.** There is no un-acknowledge operation anywhere in the API. Acknowledge only when you
have actually taken ownership.

Requires the `Acknowledge` level on `Resources` — a distinct permission from `View` and from `Manage`.
If you get `1403`, that is the level you are missing.

## 5. Add investigation notes as you go

`addAlertNoteById` — `POST /alert/alerts/{id}/note`. Notes are additive and safe to repeat.

## 6. Record the intervention

`addOpsNote` — `POST /setting/opsnotes` with `note`, `happenOnInSec` and `scopes` binding it to the device,
group or website involved. Ops Notes are what makes the next person's graph legible.

Ops Note creation is rate-limited to **100/min**, lower than the 200/min POST default.

## Failure handling

- `429` / `1429` — sleep `X-Rate-Limit-Window` seconds; no `Retry-After` is sent.
- `1403` on ack — role lacks `Resources: Acknowledge`.
- `1404` — the alert cleared and aged out between your list call and your ack.
