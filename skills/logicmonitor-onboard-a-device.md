---
name: logicmonitor-onboard-a-device
description: >-
  Add a resource to LogicMonitor monitoring — pick a healthy Collector, pick a STATIC resource group,
  create the device, set its properties, and confirm DataSources applied.
api: LogicMonitor REST API v3
base_url: https://{account}.logicmonitor.com/santaba/rest
spec: openapi/logicmonitor-rest-api-v3-swagger.json
generated: '2026-08-29'
method: generated
source: >-
  Grounded in operationIds and x-minimum-permissions read from the published v3 Swagger, plus
  https://www.logicmonitor.com/support/rest-api-developers-guide/
operations:
  - getCollectorList
  - getDeviceGroupList
  - addDevice
  - addDeviceProperty
  - getDeviceDatasourceList
  - patchDevice
permissions:
  - 'Settings -> Collectors: View'
  - 'Resources: Manage'
---

# Onboard a device into LogicMonitor

Creating a device requires two numeric ids you do not have yet — a Collector and a resource group — so the
first two calls are lookups, not guesses.

## Before you start

- Send every request to `https://{account}.logicmonitor.com/santaba/rest`, where `{account}` is the
  customer's own portal subdomain. There is no shared host.
- Add `X-Version: 3` (or `?v=3`). Without it you may be served an older version.
- Authenticate with `Authorization: Bearer <token>`, or LMv1. If you sign LMv1, exclude query parameters
  (`filter`, `fields`, `size`, `offset`) from the signed resource path and keep the timestamp within
  30 minutes of server time.
- Your token needs `Resources: Manage` and `Settings -> Collectors: View`.

## 1. Find a Collector that is up

`getCollectorList` — `GET /setting/collector/collectors?filter=isDown:false&size=50`

Read `id` and `hostname`. A Collector that is down cannot poll the new device; error `1073` means the
Collector is not active.

## 2. Find a STATIC resource group

`getDeviceGroupList` — `GET /device/groups?filter=appliesTo:""&size=50`

This filter is the whole point of the step. A group with a non-empty `appliesTo` is DYNAMIC: membership is
rule-evaluated and a device cannot be assigned to it. Pick a group whose `appliesTo` is empty and note its
`id`.

## 3. Create the device

`addDevice` — `POST /device/devices`

Required in the body: `name` (the hostname or IP LogicMonitor will poll), `displayName`,
`hostGroupIds` (comma-separated ids from step 2), `preferredCollectorId` (the id from step 1).

There is **no idempotency key on this API**. If the call times out, do not blind-retry — re-run
`getDeviceList` with `filter=name:"<name>"` first. A duplicate create returns `1409 / HTTP 409`
"record already exists", which is the only duplicate protection you get.

## 4. Set the properties monitoring needs

`addDeviceProperty` — `POST /device/devices/{deviceId}/properties`

Credentials and classification properties (`snmp.community`, `system.categories`, custom tags). Property
names must start with a letter or you get error `14003`.

## 5. Confirm DataSources applied

`getDeviceDatasourceList` — `GET /device/devices/{deviceId}/devicedatasources`

An empty list right after creation is normal; discovery has not run yet. Re-poll rather than re-creating
the device.

## 6. Adjust without replacing

`patchDevice` — `PATCH /device/devices/{id}` changes only the fields you send. `PUT /device/devices/{id}`
replaces the whole object and will silently drop anything you omit. Prefer PATCH.

## Failure handling

- `429` / `1429` — read `X-Rate-Limit-Window` and sleep that many seconds. **No `Retry-After` header is
  sent.** `POST` defaults to 200/min per account.
- `1403` — the token's role lacks the permission; `1401` — bad signature or expired timestamp.
- `1404` on a group or collector id — re-resolve it, ids are not portable between portals.

## Undoing this

`DELETE /device/devices/{id}` moves the resource to the **Recently Deleted** folder, where it can be
restored for **seven days** — but only through the LogicMonitor UI. There is no restore operation in this
API. Never call `DELETE /device/groups/{id}` with Cascade Delete to undo an onboarding: it removes member
resources across the entire account, including resources that belong to other groups.
