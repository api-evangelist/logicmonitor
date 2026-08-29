---
name: logicmonitor-schedule-maintenance
description: >-
  Suppress LogicMonitor alerting for a planned change window by creating the right kind of SDT, then end
  it early when the work finishes.
api: LogicMonitor REST API v3
base_url: https://{account}.logicmonitor.com/santaba/rest
spec: openapi/logicmonitor-rest-api-v3-swagger.json
generated: '2026-08-29'
method: generated
source: Grounded in operationIds and x-minimum-permissions read from the published v3 Swagger.
operations:
  - getDeviceList
  - getDeviceGroupList
  - addSDT
  - getSDTList
  - getSdtById
  - patchSdtById
  - deleteSdtById
permissions:
  - 'Resources: View'
  - 'Resources: Manage'
---

# Schedule downtime around a change window

An SDT (Scheduled Down Time) is the one write on this API that is cleanly reversible: you create it, and
you can delete it. Use it instead of disabling alerting on the object, which is not.

## 1. Resolve what you are silencing

`getDeviceList` — `GET /device/devices?filter=displayName~"<substring>"&fields=id,displayName`
or `getDeviceGroupList` — `GET /device/groups?filter=name~"<substring>"`

Ids are opaque integers scoped to this portal. Never reuse an id from another account.

## 2. Choose the SDT type deliberately

`POST /sdt/sdts` is polymorphic on `sdtType`. Pick the narrowest one that covers the work:

| type | silences | id field |
|---|---|---|
| `ResourceSDT` | one device | `deviceId` |
| `ResourceGroupSDT` | a device group | `deviceGroupId` |
| `CollectorSDT` | everything a Collector polls | `collectorId` |
| `WebsiteSDT` / `WebsiteGroupSDT` | a synthetic check or its group | `websiteId` / `websiteGroupId` |
| `DeviceDataSourceSDT` | one DataSource on one device | `deviceId` + `dataSourceId` |
| `DeviceDataSourceInstanceSDT` | a single instance | instance ids |

A `CollectorSDT` during a two-host patch window silences every device that Collector polls. Prefer
`ResourceSDT` per host.

## 3. Create it

`addSDT` — `POST /sdt/sdts`

- One-off: `type: oneTime` with `startDateTime` and `endDateTime` in **epoch milliseconds**.
- Recurring: `daily` / `weekly` / `monthly` / `monthlyByWeek` with `hour`, `minute`, `endHour`,
  `endMinute`, `duration`, plus `weekDay` / `monthDay` / `weekOfMonth`.
- Always set `comment` to the change reference. It is the only field a human reading the SDT list later
  can use to tell whether the window is still legitimate.

Set a bounded end time. An SDT with an over-long window is indistinguishable from monitoring that was
quietly switched off.

## 4. Verify

`getSDTList` — `GET /sdt/sdts?filter=isEffective:true` shows what is currently suppressing alerts.
`getSdtById` — `GET /sdt/sdts/{id}` for one.

## 5. End it when the work finishes

`deleteSdtById` — `DELETE /sdt/sdts/{id}` ends the downtime immediately.
`patchSdtById` — `PATCH /sdt/sdts/{id}` extends or shortens it without recreating it.

Deleting an SDT restores alerting from that moment forward. It does **not** retroactively raise alerts that
were suppressed while the window was open — those events are gone. That is the limit of the reversal.

## Failure handling

- `1403` — role lacks `Resources: Manage`.
- `1404` — the target device, group or collector id no longer resolves.
- `429` / `1429` — sleep `X-Rate-Limit-Window` seconds.
