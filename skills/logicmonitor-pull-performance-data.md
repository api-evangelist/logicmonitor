---
name: logicmonitor-pull-performance-data
description: >-
  Walk LogicMonitor's four-level telemetry hierarchy — device, DataSource, instance, datapoint — and pull
  time-series performance data for a resource.
api: LogicMonitor REST API v3
base_url: https://{account}.logicmonitor.com/santaba/rest
spec: openapi/logicmonitor-rest-api-v3-swagger.json
generated: '2026-08-29'
method: generated
source: Grounded in operationIds and x-minimum-permissions read from the published v3 Swagger.
operations:
  - getDeviceList
  - getDeviceDatasourceList
  - getDeviceDatasourceInstanceList
  - getDeviceDatasourceInstanceData
  - fetchDeviceInstancesData
permissions:
  - 'Resources: View'
---

# Pull performance data out of LogicMonitor

Metrics are four levels deep and each level's id is the parameter for the next. There is no shortcut from a
device name to a metric series.

## 1. Device

`getDeviceList` — `GET /device/devices?filter=displayName:"<name>"&fields=id,displayName`
Rate limit **700/min** — higher than the 500/min GET default.

## 2. DataSources applied to that device

`getDeviceDatasourceList` — `GET /device/devices/{deviceId}/devicedatasources`

The `id` you get back is the **hdsId** used in every path below. It is not the DataSource definition id from
`/setting/datasources` — those are different objects and swapping them yields `1404`.

## 3. Instances of that DataSource

`getDeviceDatasourceInstanceList` — `GET /device/devices/{deviceId}/devicedatasources/{hdsId}/instances`

Rate limit **500/min**. One instance per interface, disk, volume, or whatever the DataSource discovers.

## 4. The data

`getDeviceDatasourceInstanceData` —
`GET /device/devices/{deviceId}/devicedatasources/{hdsId}/instances/{id}/data?start=<epoch>&end=<epoch>`

`start` and `end` are epoch seconds. Data comes back newest-first. Ask for the window you need — a wide
range on a busy instance returns `1413` / HTTP 413 "request entity too large", and the fix is a narrower
window, not a retry.

## Bulk alternative — and its trap

`fetchDeviceInstancesData` — `POST /device/instances/datafetch` fetches many instances in one call.

Its rate limit is **5 per minute**, the tightest on the entire API. Budget for it: a loop that calls it once
per device will be throttled within seconds. Batch the instance list into as few calls as possible.

## Paging and shaping

- `size` (default 50, max 1000) and `offset`; no cursor, no next-link.
- `fields` takes a comma-separated camelCase list. snake_case is rejected here even where the Python SDK
  accepts it elsewhere.
- Responses wrap results in a `*PaginationResponse` with `total`, `items` and `searchId`.

## Failure handling

- `429` / `1429` — read `X-Rate-Limit-Window` and sleep. No `Retry-After`.
- `1413` — narrow the time range.
- `1101` — query timed out; narrow the range or the instance set.
- `1073` — the Collector is down, so recent data will be missing regardless of what you ask for.
